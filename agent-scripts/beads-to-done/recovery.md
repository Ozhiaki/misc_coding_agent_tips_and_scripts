# Recovery

When the JSONL and DB disagree, when JSONL has git conflict markers, when a claim is stuck because past-you crashed, or when `br doctor` finds something that needs fixing. This file is the playbook.

Two principles before the procedures:

- **Diagnose before acting.** `br sync --status` is cheap and read-only. Run it first. Many "recovery" situations turn out to be a one-command fix.
- **`--force` is not a fix; it's a choice.** Every guard exists to prevent a specific data-loss mode. Reading the guard's message and understanding what it's protecting takes 10 seconds and prevents the worst mistakes.

## The `br sync --status` decision tree

Start here for anything that smells like a JSONL/DB problem:

```bash
br sync --status
```

What it returns and what to do:

```
"in sync"
  → No action.

"DB ahead" (DB has mutations not yet in JSONL)
  → br sync --flush-only

"JSONL ahead" (JSONL has lines the DB doesn't, e.g., after git pull)
  → br sync --import-only

"diverged" (both sides have new work since last sync)
  → br sync --merge   (see "Three-way merge" below)
  → If that fails: choose --force-db or --force-jsonl deliberately

"JSONL unparseable" or "conflict markers in JSONL"
  → see "JSONL conflict markers" below.
  → --force does NOT bypass this guard.
```

The `--status` output has more detail in `--json` mode:

```bash
br sync --status --json
```

Use the JSON when you need to read the exact state rather than the human-readable summary.

## Three-way merge

The common diverged case: you made local mutations (in DB, auto-flushed to JSONL), then `git pull` brought in someone else's mutations to JSONL. Both sides have legitimate work.

```bash
br sync --status        # confirm "diverged"
br sync --merge         # three-way merge using common ancestor
br sync --status        # verify resolved
```

`--merge` uses the last-known-sync state as the common ancestor and combines both sides without preferring one. It refuses (with `SYNC_CONFLICT`, exit code 6) if it can't reconcile — usually because both sides edited the same field of the same issue.

If `--merge` refuses, you choose:

```bash
br sync --force-db        # DB wins; JSONL is rewritten from DB
br sync --force-jsonl     # JSONL wins; DB is rebuilt from JSONL
```

These are destructive on the losing side. Snapshot before forcing if you're not sure which side has the work you care about:

```bash
cp .beads/issues.jsonl /tmp/issues.jsonl.bak       # if forcing DB to win
sqlite3 .beads/beads.db ".backup /tmp/beads.db.bak" # if forcing JSONL to win
```

## JSONL conflict markers

When `git pull` introduces merge-conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`) into `.beads/issues.jsonl`, `br sync --import-only` refuses with `CONFLICT_MARKERS`. **`--force` does not bypass this**, on purpose — importing a file with conflict markers is guaranteed data corruption.

Resolve manually:

1. Open `.beads/issues.jsonl` in an editor.
2. Each JSONL line is one issue. For each conflict block, usually you keep **both** sides' new lines and remove only the marker lines (`<<<<<<<`, `=======`, `>>>>>>>`). It's rare that the two sides edited the same issue line — more commonly, they each added different issues, and you want all of them.
3. If both sides did edit the same issue, you'll have to pick one or hand-merge the JSON. The line is one self-contained JSON object; if you're not comfortable hand-merging JSON, keep the side with the work you don't want to lose and re-do the other side's work post-import.
4. Save and verify parseability with a dry run:

    ```bash
    br sync --import-only --dry-run
    ```

5. If clean, import for real:

    ```bash
    br sync --import-only
    ```

If you can't make sense of the conflict, the nuclear option is to declare the DB canonical and regenerate JSONL from it:

```bash
git checkout HEAD -- .beads/issues.jsonl   # discard the conflicted JSONL
br sync --force-db                          # regenerate JSONL from DB
```

This loses whatever was in the JSONL side that the DB didn't have. Use only when the DB definitely has everything you want.

## Same-agent stuck claim

The execution-time scenario this skill cares about: past-you (same agent identity) claimed an issue, crashed or was abandoned mid-work, and now the claim is stuck. `br ready` won't show the issue (it's already claimed), and you can't simply re-claim because the issue thinks it's still assigned.

Note: this is **not** the swarm "abandoned by another agent" case. That's `br coordination status` territory, which this skill explicitly doesn't cover. The same-agent case is simpler.

The fix:

```bash
# 1. Read what past-you left in notes
br show <id>

# 2. Add an audit comment recording the reclaim
br comments add <id> "Same-agent reclaim: previous session of this agent did not close out. Reclaiming to resume."

# 3. Force re-claim
br update <id> --claim --force
```

The `--force` is what gets you past the "already assigned" check. The audit comment keeps the trail honest: a future read of `br audit log <id>` plus `br comments list <id>` will show what happened.

If past-you's notes are empty or sparse, treat the issue as if you've never seen it — re-read `description`, `acceptance`, and the parent epic's `agent_context`. Don't assume past-you got far; you might be starting closer to scratch than you think.

### When the stuck claim *isn't* yours

If `br list --status in_progress` shows an issue claimed by a different actor — even if you think that actor is also you under a different name (e.g., a different `BR_AGENT_NAME` from a previous setup) — slow down. The right action depends on what's going on:

- **If it really is past-you with a different actor name**: switch to that actor for the reclaim (or update the actor), add an audit comment recording the identity collapse, then reclaim. This skill's solo-agent assumption is identity-based, not literal-actor-string-based.
- **If it's a different agent or human teammate**: this skill is the wrong skill. You're in multi-agent territory, which is explicitly out of scope. Coordinate out-of-band.

## History backups

`br sync --flush-only` (and the auto-flush path) writes a backup to `.beads/.br_history/` before overwriting JSONL. Manage with:

```bash
br history list                              # most-recent-first
br history diff <backup-file>                # diff a backup against current JSONL
br history restore <backup-file>             # restore a backup as current JSONL
br history prune --keep N --older-than DAYS  # bounded retention
```

`br history restore` overwrites `.beads/issues.jsonl` but does **not** touch the DB. After restoring, you almost always want to reconcile:

```bash
br history restore <backup-file>
br sync --import-only          # or --force-jsonl if the DB has stale state
```

A reasonable retention default: `br history prune --keep 20 --older-than 60`. Keeps the most recent 20 backups and additionally prunes anything older than 60 days. Adjust to taste.

## DB corruption / rebuild

When the DB is corrupted (errors with `DATABASE_ERROR` or `SCHEMA_MISMATCH` that migration can't fix, or the SQLite file is unreadable) and the JSONL is intact:

```bash
# Move the broken DB aside (don't delete; you may want it for forensics)
mv .beads/beads.db /tmp/beads.db.broken

# Rebuild from JSONL
br sync --rebuild
# (or, equivalently for fresh setups: br sync --import-only)
```

`--rebuild` regenerates the DB schema and replays every JSONL line. After it completes, verify with `br ready` and `br list --status open`.

## `br doctor`

`br doctor` is the scoped-repair facility for problems that aren't simple sync issues — orphaned dependency edges, schema migrations, configuration drift, gitignore gaps. It has its own exit-code dictionary distinct from ordinary `br` commands, so a wrapper script must interpret doctor exit codes against the doctor table.

```bash
br doctor                  # diagnose only (default)
br doctor diagnose         # explicit diagnose
br doctor repair           # diagnose + apply fixes (writes per-run backups)
br doctor ls               # list past doctor runs
br doctor undo <run-id>    # roll back a repair from its backup snapshot
br doctor undo latest      # roll back the most recent repair
br doctor explain <finding-id>  # human explanation of a finding
```

The flow when something is off and you don't know what:

1. **Diagnose**: `br doctor` (or `br doctor --format json` to get structured findings).
2. **Read each finding**: `br doctor explain <finding-id>` for context.
3. **Decide whether to repair**: if the findings are things you'd want fixed, `br doctor repair`. Otherwise act on them individually.
4. **If repair did something you regret**: `br doctor undo latest` (or `br doctor undo <run-id>`). Doctor writes per-run backups before mutating, so undo is robust.

### Doctor exit codes (different from ordinary)

A scripting gotcha worth knowing: `br doctor`'s exit codes are not the same as ordinary `br` commands'. The numbers overlap; the meanings don't.

| Exit | Ordinary `br` | `br doctor` |
|---|---|---|
| 0 | Success | Healthy (no findings) |
| 1 | Internal error | FindingsPresent (diagnose-only found problems) |
| 2 | Database error | FixPartial (some fixes applied, some failed) |
| 3 | Issue error | FixFailedRolledBack |
| 4 | **Validation** | **RefusedUnsafe** |
| 5 | Dependency | ConcurrencyLost |
| 6 | Sync/JSONL | OnlineRequired |

The collision that matters most is exit code **4**: ordinary `br` validation error vs doctor's "refused to act for safety reasons." A wrapper that retries on "validation errors" must not retry doctor exit code 4 — that's a refusal, not a fixable input problem.

If you're scripting around `br`, gate exit-code interpretation on which command you ran. For interactive use, just read the message — it'll tell you which table applies.

## When the safety guards fire

These messages are common; here's what to do for each.

### "Refusing to export empty database..."

DB has 0 issues, JSONL has N. Almost certainly means the DB needs to be imported from JSONL, not the other way around.

```bash
br sync --import-only      # the usual fix
# OR, only if you really did intend an empty export:
br sync --flush-only --force
```

### "Refusing to export stale database..."

JSONL has issues the DB doesn't. The DB needs to catch up before it can flush.

```bash
br sync --import-only      # the usual fix
# OR, only if you intend to lose the JSONL-only issues:
br sync --flush-only --force
```

### "Merge conflict markers detected..."

JSONL has unresolved git conflict markers. Resolve them manually (see "JSONL conflict markers" above). `--force` does not bypass this guard.

### "Cycle detected" (on `br dep add`)

The dependency you're trying to add would create a cycle. Two options:

- Reconsider the dependency direction. If A blocks B and B blocks A, one of those is wrong.
- Re-shape the issues. If two issues genuinely have a circular requirement, they probably want to be merged or split differently.

`br dep tree <id>` shows the current graph; useful for figuring out where the cycle would close.

## When everything is broken

A pragmatic last-resort sequence when the workspace is in a state you can't quickly understand:

```bash
# 1. Snapshot everything
cp -r .beads /tmp/beads-broken-$(date +%Y%m%d-%H%M%S)

# 2. Use git to recover JSONL to a known-good state
git log --oneline .beads/issues.jsonl     # find a good commit
git checkout <good-commit> -- .beads/issues.jsonl

# 3. Rebuild DB from JSONL
mv .beads/beads.db /tmp/beads-db-broken-$(date +%Y%m%d-%H%M%S).db
br sync --rebuild

# 4. Verify
br sync --status
br ready
br list --status open
```

You'll lose any DB-only mutations between the chosen good commit and now. But you'll have a working workspace and a snapshot of the broken one to forensic-debug later if needed.

## Checklist for any "something feels off" moment

1. **`br sync --status`** first. Read the actual state, don't guess.
2. **Match the message to the procedure above.** Each guard has a specific recovery.
3. **Snapshot before `--force`** anything destructive: `cp .beads/issues.jsonl /tmp/...` and/or `sqlite3 .beads/beads.db ".backup /tmp/..."`.
4. **For stuck same-agent claims**: read past-you's notes, audit-comment, force re-claim.
5. **For unfamiliar `br doctor` situations**: diagnose, explain each finding, then decide on repair. Use `br doctor undo` if the repair surprised you.
6. **`--force` is a deliberate choice**, never a knee-jerk retry. If you find yourself reaching for it, read the message you're bypassing first.
