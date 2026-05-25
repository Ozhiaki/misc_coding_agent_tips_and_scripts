# Closing and Sync

The closing half of the loop. Verify acceptance, write a traceable close reason, sync the JSONL/DB pair, and do the git half. Work is not done until `git push` succeeds.

## What "done" means

Done is not "I wrote the code." Done is the full sequence:

1. Every acceptance criterion is verifiably met.
2. The issue is closed with `br close <id> --reason "<traceable>"`.
3. The JSONL is current (auto-flush usually handles this; `br sync --flush-only` is the idempotent check).
4. The `.beads/` changes are staged, committed, and pushed.

Skipping any step looks fine in the moment and fails one session later, when a future agent (or the rest of the team) can't tell the work was actually completed. Specifically:

- Skip step 2 → the issue still shows `in_progress`. Future `br ready` won't surface it (still claimed), future `br list --status in_progress` will think it's still in flight.
- Skip step 3 → the close is in the DB but not the JSONL. After a `git pull` somewhere else, the close is silently lost.
- Skip step 4 → the close is local to your machine. Anyone else's `br ready` still thinks the work is open.

The full sequence is short; just commit to running it.

## Step 1: verify acceptance

Before reaching for `br close`, re-read the issue:

```bash
br show <id>
```

Read the `acceptance_criteria` field specifically. For each criterion, satisfy yourself it's met. If you can't honestly tick a criterion off:

- **The criterion is unverifiable as written** → branch 2 of [discovery.md](discovery.md); fix the criterion (with a `br comments add` recording why), then re-check.
- **The criterion is verifiable but unmet** → there's more work to do; don't close yet.
- **The criterion describes a behavior you didn't implement because of a mid-work pivot** → either the pivot was wrong (go back), or the criterion is now wrong (update it before closing, with `br comments add` recording why).

The temptation to close with a criterion still unmet, especially on solo work where "nobody will check," is real. Resist. The acceptance criteria are the contract you wrote with future-you; if you erode it, future-you stops trusting it.

If acceptance criteria are missing entirely on an issue you're about to close, you've got an under-specified issue. Two options:

- **Backfill acceptance criteria honestly before closing** (`br update <id> --acceptance-criteria "..."` then close).
- **Note the gap explicitly in the close reason** ("Acceptance criteria were not set; closing based on observed behavior: <list>").

The first option is preferred. It costs 30 seconds and leaves the issue in a state where the audit trail makes sense.

## Step 2: close with a traceable reason

The `--reason` is the last thing a future agent sees about this issue. Make it a breadcrumb back to the code.

```bash
br close <id> --reason "<what changed + commit ref>" --suggest-next --json
```

### What goes in `--reason`

The reason should answer two questions:

1. **What changed?** A one-line summary of the work, specific enough that a future agent could find it without re-reading the issue.
2. **Where to find it?** A commit ref (preferably the SHA prefix; PR number is also fine) so the diff is one click away.

Anti-pattern → fixed:

| ❌ Bad | ✅ Good |
|---|---|
| `--reason "Done"` | `--reason "Replaced log/syslog with log/slog in internal/logging — commit a1b2c3d"` |
| `--reason "Implemented"` | `--reason "Added Sensitive bool to SecretObject with UnmarshalJSON backfill — commit f4e5d6c"` |
| `--reason "Fixed the bug"` | `--reason "Fixed race in credential rotation: added sync.Mutex around read-check-rotate in rotator.go — commit 7890abc"` |
| `--reason "See PR"` | `--reason "Migrated session middleware to interface-based design — PR #142, commit 3def123"` |

### `--suggest-next`

`--suggest-next` on close returns a list of issues that just became ready because *this* issue's close unblocked them. Highly worth using for chained work:

```bash
$ br close br-42 --reason "Refactored session middleware to interface-based design — commit a1b2c3d" --suggest-next --json
{
  "closed": ["br-42"],
  "skipped": [],
  "unblocked": [
    {"id": "br-43", "title": "Migrate admin tools to new session interface", "priority": 1},
    {"id": "br-44", "title": "Update CLI auth command to new session interface", "priority": 2}
  ]
}
```

If `unblocked` is non-empty, you have a natural next-pick: take the highest-priority item, run through [claim-and-work.md](claim-and-work.md), and continue.

If `unblocked` is empty, the work this issue was unblocking either doesn't exist or is itself blocked elsewhere. Drop back to `br ready` for the broader queue.

### Closing multiple issues at once

When you finish work that closes several related issues simultaneously (rare but it happens — e.g., a refactor that satisfies three small issues at once):

```bash
br close br-42 br-43 br-44 --reason "Unified session interface across middleware, admin, and CLI — commit a1b2c3d"
```

Same `--reason` applies to all. If the issues legitimately had different outcomes, close them separately with different reasons. Don't paper over different work with one shared close reason — that's the same anti-pattern as merging two issues into one diff.

### Closing as obsolete or won't-fix

Not every close is a "we did the work" close. Two other shapes worth knowing:

- **Obsolete** (the work is no longer relevant): `br close <id> --reason "Obsolete: replaced by br-99 which takes a different approach"` or `--reason "Obsolete: behavior already provided by commit abc1234 (unrelated work)"`.
- **Won't fix** (we've decided not to do this): `br close <id> --reason "Won't fix: per user, the cost of the migration outweighs the benefit; behavior accepted as-is"`.

These are still traceable reasons — they explain why the issue is closed, not just that it is.

## Step 3: sync to JSONL

`br` auto-flushes mutating commands to JSONL by default, so after `br close` the JSONL is normally current. But before staging for commit, run the idempotent final check:

```bash
br sync --flush-only
```

This is a no-op when the JSONL is current, so the cost of running it is near zero. The value of running it is preventing a "JSONL didn't get the close" failure mode that would otherwise surface as the commit not actually containing the close.

If you have auto-flush disabled (rare; see project config), or you used `--no-auto-flush` on the close, this step isn't optional — it's required.

### What `--flush-only` actually does

It exports the SQLite DB's current state to `.beads/issues.jsonl`, atomically (temp file + rename), with a backup snapshot written to `.beads/.br_history/` first. The export refuses if either safety guard triggers:

- **Empty DB guard**: refuses to overwrite N records in JSONL with 0 records from an apparently-empty DB. Override: `--force` (use only when you've deliberately cleared the DB).
- **Stale DB guard**: refuses to export if the JSONL contains issues the DB doesn't. Override: `--force` (use only when you know the DB is canonical).

Neither guard should fire in a normal close flow. If one does, **stop and read** [recovery.md](recovery.md). Don't `--force` your way past a safety guard you don't understand.

## Step 4: the git half

`br` never runs git. Every `git` command in this section is something you run explicitly.

```bash
git add .beads/
git commit -m "Close br-42: refactor session middleware to interface-based design"
git push
```

A few details:

- **`git add .beads/`** stages `.beads/issues.jsonl` (and any other `.beads/` files like `config.yaml` if it changed). It does **not** stage `.beads/beads.db` if your `.gitignore` is set up right (the DB is local, the JSONL is canonical). If you see `.beads/beads.db` in the diff, your `.gitignore` is missing it; fix that.
- **Commit message** is up to you, but a useful convention is `Close <id>: <short summary>` — it mirrors the close `--reason` and makes the git log scannable. If multiple issues closed in this commit, list them.
- **`git push`** is the actual ship moment. Work is not done before this.

### What if push fails

Push can fail for prosaic reasons (someone else pushed first, branch protection, network). The recovery is the same as any git push failure:

```bash
git pull --rebase    # or whatever your team's flow is
br sync --import-only    # in case the pull brought in new JSONL content
br sync --status         # confirm we're in sync
git push
```

Pulling can introduce conflicts in `.beads/issues.jsonl`. If git puts conflict markers in the JSONL, **don't run `br sync --import-only` until they're resolved** — there's a guard that will refuse, and `--force` won't bypass it. See [recovery.md](recovery.md) under "JSONL conflict markers."

## A worked close

Putting it together. You've finished `br-42`. The commit is local; you haven't closed the issue yet.

```bash
# 1. Verify acceptance
$ br show br-42
# (read the acceptance criteria, confirm each is met)

# 2. Close with a traceable reason
$ br close br-42 --reason "Refactored session middleware to interface-based design (Session interface + SessionLegacy compat shim) — commit a1b2c3d" --suggest-next --json
{
  "closed": ["br-42"],
  "skipped": [],
  "unblocked": [
    {"id": "br-43", "title": "Migrate admin tools to new session interface", "priority": 1}
  ]
}

# 3. Idempotent sync check
$ br sync --flush-only
Already in sync.

# 4. Git half
$ git add .beads/
$ git commit -m "Close br-42: refactor session middleware to interface-based design"
[main a1b2c3d] Close br-42: refactor session middleware to interface-based design
 2 files changed, 14 insertions(+), 3 deletions(-)
$ git push
# ...push output...

# 5. Decide next move
# br-43 is unblocked. Pick it up via the claim flow.
$ br show br-43
# ...
$ br update br-43 --claim
```

Five steps. The whole sequence takes a couple of minutes. Skipping any of them takes much longer to recover from.

## The JSONL/DB model, briefly

You don't need a deep model to do the close right, but it helps to know what the two files are.

- **`.beads/issues.jsonl`** is the canonical, version-controlled record. One issue per line, JSON-encoded, committed to git. **Never hand-edit.** Every mutation flows through `br`.
- **`.beads/beads.db`** is a SQLite file representing the current working state. Local-only (gitignored). Rebuilt from JSONL on `br sync --import-only` or `br sync --rebuild`.

The pair is kept in sync by `br sync`:

| When | Command | What it does |
|---|---|---|
| After `git pull` | `br sync --import-only` | Update DB from JSONL. |
| Before commit | `br sync --flush-only` | Export DB to JSONL (auto-flush usually has this done; this is the idempotent check). |
| Status check | `br sync --status` | Report which side is ahead, without mutating. |
| Both sides advanced | `br sync --merge` | Three-way merge. See [recovery.md](recovery.md). |

The model is dual-edge sword: it lets multiple developers/agents commit issue mutations to git like any other change, but it requires explicit syncing because `br` never runs git on your behalf.

## Mistakes the discipline prevents

| Mistake | Cost |
|---|---|
| Closed without verifying acceptance | Future audit finds work was closed but not actually done; trust in the tracker erodes. |
| Vague `--reason` ("Done", "Fixed") | Future agent has no breadcrumb to the code; has to dig through git log. |
| Closed but didn't `git push` | Issue closed locally, still open everywhere else; team works on already-done items. |
| Pushed without `br sync --flush-only` and auto-flush disabled | JSONL stale; team sees the issue still open. |
| `git add . -A` swept in `.beads/beads.db` | DB committed, churns the diff on every operation, eventually causes merge headaches. |
| Force-overrode a safety guard without reading | Data loss; JSONL truncated or rebuilt incorrectly. |

The fix in every case is the five-step sequence, run honestly.

## Checklist

For every close:

1. **`br show <id>`** — read the acceptance criteria, confirm each is met.
2. **`br close <id> --reason "..." --suggest-next --json`** — reason includes *what* and *commit ref*; `--suggest-next` surfaces follow-on work.
3. **`br sync --flush-only`** — idempotent check, near-zero cost, prevents the silent-skip failure mode.
4. **`git add .beads/` → `git commit -m "..."` → `git push`** — the work isn't done until push succeeds.
5. **Pick the next item from `--suggest-next` or `br ready`**, and resume the loop.
