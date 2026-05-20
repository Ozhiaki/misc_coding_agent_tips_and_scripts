# Session Protocol

> The preflight checklist: what an agent does at session start, during work, and before saying "done."

## Session Start

```bash
br sync --status                    # Check if import needed
br sync --import-only               # Import any JSONL changes from git pull
br ready --json                     # Find unblocked work
br list --status in_progress --json # Check for previously claimed work
```

If `in_progress` issues exist, run `br show <id>` and read the notes before doing anything else. Another agent (or you, pre-compaction) left context there.

## Claim → Work → Update Loop

```bash
br show <id>                        # Read full context before coding
br update <id> --claim              # Atomically take ownership
# ... do the work ...
br update <id> --notes "COMPLETED: ... / IN_PROGRESS: ... / NEXT: ..."
```

**Update notes at milestones**, not just at the end. If context compacts mid-session, the notes are the only record of what happened.

### Notes Discipline

`br` uses `--notes` which **overwrites** the existing value. To preserve history, read the current notes first and include them:

```bash
# Read existing notes
br show <id> --json | jq -r '.notes'

# Overwrite with old + new combined
br update <id> --notes "Previous notes here...
---
2026-03-15: New session. Completed X, discovered Y."
```

This is a footgun. Always read-before-write. Losing session notes is losing institutional memory.

## Session Close

Before saying "done," complete **all** steps:

```bash
br close <id> --reason "Implemented OAuth PKCE flow — see abc1234"
# Mutations auto-flush. This is the idempotent final check:
br sync --flush-only
git add .beads/
git commit -m "Close <id>: short summary"
git push
```

The auto-flush behavior means JSONL is usually current after `br close`. The
explicit `--flush-only` exists for parity with the historical Go workflow
and as recovery insurance.

**Work is NOT done until `git push` succeeds.**

### Close Reasons

Include both *what changed* and *where to find it*:

```bash
# ❌ Vague
br close br-42 --reason "Implemented"

# ✓ Traceable
br close br-42 --reason "Added PKCE to OAuth login flow — commit a1b2c3d"
```

The close reason is the last thing a future agent sees. Make it a breadcrumb back to the code.

### Chained Work

Use `--suggest-next` to see what you just unblocked:

```bash
br close <id> --reason "..." --suggest-next --json
```

Then claim the next highest-priority unblocked issue and continue.

## Quick Reference

| Phase | Commands |
|-------|----------|
| Start | `sync --import-only` → `ready` → `show` |
| Claim | `update <id> --claim` |
| Work | `update <id> --notes "..."` (at milestones) |
| Close | `close <id> --reason "what + commit ref"` |
| Flush | `sync --flush-only` (idempotent final check) → `git add/commit/push` |
| Next | `close --suggest-next` or `ready` |
