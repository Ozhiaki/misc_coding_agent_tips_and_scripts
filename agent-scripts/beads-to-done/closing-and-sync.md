# Closing and Sync

The closing half of the loop. Verify acceptance, clean up notes, write a traceable close reason, commit in the right order, sync the JSONL/DB pair, and ship. Work is not done until `git push` succeeds.

## What "done" means

Done is not "I wrote the code." Done is the full sequence:

1. Every acceptance criterion is verifiably met.
2. Notes are cleaned up — no stale `IN_PROGRESS` / `NEXT` left over from mid-work snapshots.
3. The **implementation commit** is in (so the close `--reason` has a real SHA to cite).
4. The issue is closed with `br close <id> --reason "<traceable, includes SHA>"`.
5. The JSONL is current (auto-flush usually handles this; `br sync --flush-only` is the idempotent check).
6. The **tracker commit** stages `.beads/` and ships it.

Skipping any step looks fine in the moment and fails one session later, when a future agent (or the rest of the team) can't tell the work was actually completed. Specifically:

- Skip step 2 → notes lie. A future glance at `br show <id>` shows an issue marked closed but with `IN_PROGRESS / NEXT: ...` in notes; the next agent can't tell whether something was left undone.
- Skip step 3 (close before commit) → the close `--reason` has no SHA to cite, or cites a hand-wavy "see latest commit," which doesn't survive a `git log` weeks later.
- Skip step 4 → the issue still shows `in_progress`. Future `br ready` won't surface it (still claimed), future `br list --status in_progress` will think it's still in flight.
- Skip step 5 → the close is in the DB but not the JSONL. After a `git pull` somewhere else, the close is silently lost.
- Skip step 6 → the close is local to your machine. Anyone else's `br ready` still thinks the work is open.

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

## Step 2: clean up notes

If you followed [notes-discipline.md](notes-discipline.md), the issue's `notes` field has been updated at milestones during work and almost certainly contains text like:

```
COMPLETED: ...
IN_PROGRESS: implementing UnmarshalJSON for legacy backfill.
NEXT: tests.
BLOCKERS: none.
```

After the work is actually done, that text is misleading — there's no `IN_PROGRESS` or `NEXT` left. Closing without cleaning up means the closed issue's notes contradict its status. A future search for "what's in flight?" via `br show` could surface the issue and confuse the reader.

The fix is 15 seconds. Either:

**Option A: rewrite notes as a final state.** Replace handoff-shaped text with a brief done-state summary:

```bash
br update <id> --notes "<previous content, trimmed>
---
2026-05-25 CLOSED: <one-line summary>. See close reason for commit ref."
```

**Option B: strip the handoff markers, leave the history.** If the past session-by-session content is worth keeping (audit value), just drop the live `IN_PROGRESS` / `NEXT` / `BLOCKERS` lines. The COMPLETED log can stay.

Either is fine. The goal is just: **after closing, the notes should not contain text claiming work is in flight**. A reader of `br show <closed-id>` should see a coherent picture, not a stale handoff snapshot.

When to skip step 2: if the notes are already a clean done-state summary (e.g., the work was small and the only note you wrote was "All done"), there's nothing to clean up. Move on.

## Step 3: the implementation commit

Before `br close`, commit the implementation. This gives the close `--reason` a real SHA to cite:

```bash
git add <implementation paths>          # NOT .beads/ yet
git commit -m "<implementation summary>"
git log -1 --format=%h | cat            # capture the SHA prefix for --reason
```

Important: **don't `git add .beads/` yet**. The tracker mutations from `br close` haven't happened. Staging `.beads/` now means either (a) the close lands in a separate later commit (fine, but adds a round-trip), or (b) you forget and stage `.beads/` mid-stream, mixing tracker churn with implementation. Keep them separate.

The implementation commit message convention is up to your project. A useful pattern is to reference the issue ID without claiming closure:

```
Refactor session middleware to interface-based design

Defines Session interface in internal/auth/session.go; updates auth,
admin, and CLI call sites. Compat shim retained as SessionLegacy.

Touches br-42.
```

The "Touches br-42" line gives the audit trail a back-reference without prematurely declaring the issue closed (the actual close happens in step 4, after this commit's SHA is known).

### When step 3 doesn't apply

Some issues — particularly epics, but also tracker-only changes — close without an implementation commit. See "Epic close-out" below for the no-code case.

## Step 4: close with a traceable reason

Now the close, with the SHA from step 3:

```bash
br close <id> --reason "<what changed + commit ref>" --suggest-next --json
```

### What goes in `--reason`

The reason should answer two questions:

1. **What changed?** A one-line summary of the work, specific enough that a future agent could find it without re-reading the issue.
2. **Where to find it?** A commit ref (the SHA prefix from step 3; PR number is also fine) so the diff is one click away.

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

In **queue mode** (see SKILL.md "Session mode"), `--suggest-next` is doubly useful — it lets you stay within the same epic/dep-cluster rather than context-switching to unrelated work. Prefer the unblocked items before reaching for the wider queue.

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

## Epic close-out (no-code closes)

Epics group child issues. When every child closes, the epic itself becomes closeable — but the close looks different from a leaf close: there's no implementation commit, just verification of children and a roll-up close reason.

### When an epic is ready to close

Two conditions:

1. **All children are closed.** Verify with:
    ```bash
    br list --parent <epic-id> --status open --status in_progress --status blocked --json
    ```
    An empty `issues` array means all children are accounted for.
2. **The epic's acceptance criteria are met by the children's work taken together.** This is a judgment call you make by reading the children's close reasons against the epic's `acceptance_criteria`.

If either is false, the epic is not yet closeable. Specifically: if there's a child blocked on something outside the epic, the epic stays open until the external blocker resolves.

### The epic close-out sequence

```bash
# 1. Verify children
br list --parent <epic-id> --json \
  | jq -r '.issues[] | "\(.id) \(.status): \(.title)"'

# 2. Read the epic's acceptance criteria
br show <epic-id>

# 3. Walk each criterion against child close reasons
br list --parent <epic-id> --status closed --json \
  | jq -r '.issues[] | "\(.id): \(.close_reason)"'

# 4. Update epic notes to a clean done-state (per step 2 above)
br update <epic-id> --notes "<final summary referencing the children that delivered each acceptance criterion>"

# 5. Close with a roll-up reason citing child commit refs
br close <epic-id> --reason "<roll-up summary; cite key commit SHAs from children>" --suggest-next --json
```

The roll-up close reason is the load-bearing piece. A future agent reading just the epic's `br show` should be able to find the work without paging through every child. A good template:

> Auth refactor epic complete. Session middleware interface (br-42, a1b2c3d), admin migration (br-43, b2c3d4e), CLI migration (br-44, c3d4e5f), and integration tests (br-45, d4e5f6a). All acceptance criteria met; no regressions in coverage.

### The tracker commit for an epic close

Because there's no implementation commit, an epic close skips step 3 and goes straight to step 5 (sync) and step 6 (tracker commit). The tracker commit is the *only* commit for the epic close:

```bash
git add .beads/
git commit -m "Close <epic-id>: <one-line roll-up>"
git push
```

The commit message is fine to phrase as "Close" since the only thing being committed is the tracker mutation; there's no code change to misattribute.

### Cascading epic closes

Closing the last leaf in an epic often surfaces the epic itself in `br ready` (if it's modeled with `blocks` deps from children). And closing that epic may surface a parent epic. In queue mode this can chain three or four levels deep in one session. Do them. Don't leave an epic-close hanging because it "wasn't the original ask" — closing the just-finished epic before moving on keeps the queue honest and the audit trail clean.

## Step 5: sync to JSONL

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

## Step 6: the tracker commit

`br` never runs git. Every `git` command in this section is something you run explicitly. The tracker commit is *separate* from the implementation commit from step 3:

```bash
git add .beads/
git commit -m "Close br-42: refactor session middleware to interface-based design"
git push
```

A few details:

- **`git add .beads/`** stages `.beads/issues.jsonl` (and any other `.beads/` files like `config.yaml` if it changed). It does **not** stage `.beads/beads.db` if your `.gitignore` is set up right (the DB is local, the JSONL is canonical). If you see `.beads/beads.db` in the diff, your `.gitignore` is missing it; fix that.
- **Commit message** mirrors the close `--reason`: `Close <id>: <short summary>`. Scannable in `git log`. For multi-close, list the IDs.
- **`git push`** is the actual ship moment. Work is not done before this.

### Why two commits instead of one

Mixing tracker churn into implementation commits is bad hygiene for a few reasons:

- **Reverts**: if the implementation needs to be reverted, you don't want the tracker mutation reverted with it (the close is still valid history; only the code change needs backing out).
- **Diff readability**: code reviewers should see *what changed in the codebase* in the implementation commit, not 50 lines of JSONL churn.
- **Bisect**: `git bisect` on the implementation commit shouldn't bring along tracker state that didn't exist when the bug was introduced.

The cost of the two-commit pattern is one extra `git commit`. The benefit is durable separation.

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

Putting it together. You've finished implementing `br-42` and the code is ready to commit.

```bash
# 1. Verify acceptance
$ br show br-42
# (read the acceptance criteria, confirm each is met)

# 2. Clean up notes (current text says "NEXT: tests" but tests are done)
$ br show br-42 --json | jq -r '.notes'
2026-05-23: Initial pass...
---
2026-05-24: All callers updated, tests passing locally.
NEXT: run integration suite, then close.

$ br update br-42 --notes "2026-05-23: Initial pass...
---
2026-05-24: All callers updated, integration suite passing.
---
2026-05-25 CLOSED: Refactor complete. See close reason for commit ref."

# 3. Implementation commit
$ git add internal/auth/ admin/ cli/
$ git commit -m "Refactor session middleware to interface-based design

Defines Session interface; migrates auth, admin, and CLI callers.
SessionLegacy compat shim retained.

Touches br-42."
[main a1b2c3d] Refactor session middleware to interface-based design
 12 files changed, 247 insertions(+), 89 deletions(-)

$ git log -1 --format=%h | cat
a1b2c3d

# 4. Close with the SHA
$ br close br-42 --reason "Refactored session middleware to interface-based design (Session interface + SessionLegacy compat shim) — commit a1b2c3d" --suggest-next --json
{
  "closed": ["br-42"],
  "skipped": [],
  "unblocked": [
    {"id": "br-43", "title": "Migrate admin tools to new session interface", "priority": 1}
  ]
}

# 5. Idempotent sync check
$ br sync --flush-only
Already in sync.

# 6. Tracker commit
$ git add .beads/
$ git commit -m "Close br-42: refactor session middleware to interface-based design"
[main b2c3d4e] Close br-42: refactor session middleware to interface-based design
 1 file changed, 2 insertions(+), 2 deletions(-)
$ git push
# ...push output...

# 7. Decide next move (queue mode → br-43 is unblocked)
$ br show br-43
# ...
$ br update br-43 --claim
```

Six steps plus the optional next-pick. The whole sequence takes a couple of minutes. Skipping any of them takes much longer to recover from.

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
| Closed with stale `IN_PROGRESS` / `NEXT` in notes | Closed issue's notes contradict its status; future search for in-flight work surfaces noise. |
| Closed before the implementation commit | `--reason` has no real SHA to cite; "see latest commit" doesn't survive months. |
| Mixed implementation and tracker in one commit | Reverts entangle tracker history; diffs harder to read; bisect surfaces unrelated state. |
| Vague `--reason` ("Done", "Fixed") | Future agent has no breadcrumb to the code; has to dig through git log. |
| Closed but didn't `git push` | Issue closed locally, still open everywhere else; team works on already-done items. |
| Pushed without `br sync --flush-only` and auto-flush disabled | JSONL stale; team sees the issue still open. |
| `git add . -A` swept in `.beads/beads.db` | DB committed, churns the diff on every operation, eventually causes merge headaches. |
| Closed leaves but left a now-closeable epic open | Queue mode misses a free close; future session has to back-track to clean it up. |
| Force-overrode a safety guard without reading | Data loss; JSONL truncated or rebuilt incorrectly. |

The fix in every case is the six-step sequence (plus epic close-out when appropriate), run honestly.

## Checklist

For every close:

1. **`br show <id>`** — read the acceptance criteria, confirm each is met.
2. **Clean up notes** — strip `IN_PROGRESS` / `NEXT` / `BLOCKERS` lines that no longer apply.
3. **Implementation commit first** — `git add <implementation>` (NOT `.beads/`), `git commit`, capture the SHA.
4. **`br close <id> --reason "..." --suggest-next --json`** — reason includes *what* and the SHA from step 3.
5. **`br sync --flush-only`** — idempotent check, near-zero cost, prevents the silent-skip failure mode.
6. **Tracker commit** — `git add .beads/` → `git commit -m "Close <id>: ..."` → `git push`. The work isn't done until push succeeds.
7. **If you just closed the last leaf in an epic**, check whether the epic itself is now closeable. If yes, run the epic close-out sequence before moving on.
8. **Pick the next item** from `--suggest-next` or `br ready` (queue mode), or stop (single-issue mode).
