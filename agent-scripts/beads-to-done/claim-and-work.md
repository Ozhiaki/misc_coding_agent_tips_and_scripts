# Claim and Work

Session start, picking what to work on, claiming an issue safely, retrieving the governing context from an epic, and the read-before-claim sanity check that catches a wrong issue before you've sunk time into it.

## Session start: the orienting four

Before claiming anything, run these four. They're cheap and they prevent the most common "wait, what state is this in?" surprises.

```bash
br sync --status                       # JSONL vs DB: is one ahead?
br sync --import-only                  # if status said JSONL is ahead
br list --status in_progress --json    # is anything already claimed?
br ready --json                        # what's unblocked and waiting?
```

What each one is doing for you:

- **`br sync --status`** — Tells you whether the JSONL on disk and the DB agree. After a `git pull`, the JSONL has likely advanced and the DB hasn't caught up. `--status` says so without mutating anything.
- **`br sync --import-only`** — Pulls JSONL changes into the DB. Idempotent. Run it whenever `--status` says "JSONL ahead." If `--status` says "in sync," skip this.
- **`br list --status in_progress`** — Lists everything already claimed. **Read this before reaching for `br ready`.** A claim that's still `in_progress` from a previous session of yours is the first thing to deal with — not skip past, not abandon. See "Inheriting your own stuck claim" below.
- **`br ready`** — Lists work whose dependencies are all closed (or `discovered-from` / `related`) and that nobody has claimed yet. This is your menu.

If `--status` returns anything more interesting than "in sync," "DB ahead," or "JSONL ahead" — for example "diverged" or "conflict markers in JSONL" — stop and read [recovery.md](recovery.md). Don't paper over divergence with a `--force`.

## Inheriting your own stuck claim

You sat down, you ran `br list --status in_progress`, and there's an issue claimed by you (or by the actor you'll be using). Treat this as the highest-priority signal in the session.

**Don't ignore it.** It means a past session of yours started work and didn't finish or didn't close. The issue's `notes` field is the only record of where you left off. Read it before doing anything else:

```bash
br show <id>
```

Three things to look for in the notes:

1. **What was completed.** Did past-you finish the work but not close? Then jump to [closing-and-sync.md](closing-and-sync.md).
2. **What was in progress.** Was past-you mid-implementation? Resume from where notes say "NEXT" or "IN_PROGRESS."
3. **Whether the work was abandoned.** Did past-you discover the issue was wrong, hit a blocker, get stuck on a question for the user? Then the right action is probably *not* to resume — it's to address the underlying problem first. See [discovery.md](discovery.md).

If the notes are sparse or absent — past-you claimed and crashed without writing a snapshot — you're in the *stuck claim* case. The fix is in [recovery.md](recovery.md) under "Same-agent stuck claim." Short version: audit-comment, then re-claim with `--force`. Don't just start working under the existing claim as if nothing happened; leave a trail.

## Picking ready work

Once any inherited claims are resolved, pick from `br ready`. The output is sorted by `priority` (P0 highest) and then by readiness. In the absence of other signals, take the top one.

Useful refinements:

```bash
br ready --priority 0 --priority 1    # P0 + P1 only
br ready --type bug                   # just bugs
br ready --limit 5                    # don't print the world
br ready --json                       # machine-readable for piping/jq
```

The TOON format is the lowest-token option for scanning:

```bash
br ready --format toon
```

If `br ready` returns empty but you know there's open work, two likely causes:

- Everything open is `in_progress` or blocked. Check `br list --status open --json` and `br blocked --json`.
- A dependency graph has a closed-but-not-flushed link. Run `br sync --status` again; if mutations happened in another tool, the DB might disagree.

## Read before you claim

The single most-common avoidable mistake during execution: claiming an issue, getting halfway through, then realizing the issue was incomplete, wrong, or already done by the team in another commit.

Before `br update <id> --claim`, always run:

```bash
br show <id>
```

Scan for these four things in order:

1. **Description** — does the problem statement still match reality? If the description says "the deploy script fails on macOS" and you just confirmed the deploy script works on macOS in HEAD, the issue may already be resolved. Check git log before claiming.
2. **Acceptance criteria** — are they verifiable, specific, and still applicable? Vague criteria are a yellow flag that the issue needs sharpening before work, not during. See [discovery.md](discovery.md) under "current bead is wrong."
3. **Design** — if filled in by past-you or another agent, is the approach still valid given anything that's changed in the codebase since? A stale `design` field can lead an agent to implement against an outdated mental model.
4. **Notes** — even on an issue you've never touched, the `notes` field may contain crucial context from whoever wrote the issue. Read it.

If any of those four turn up surprises, jump to [discovery.md](discovery.md) **before** claiming. Claiming first and then renegotiating is awkward — the audit trail will show the issue was claimed by you, then immediately mutated to say "actually this is wrong." Cleaner to catch it pre-claim.

**One more flag to notice at read time:** if the issue's description or acceptance hints at a new or changed user-facing input surface — a new CLI flag, an HTTP param, a config key, an env var, a schema field — keep [input-surface-testing.md](input-surface-testing.md) in mind. You won't fully know the grammar until you've picked the implementation shape, but noticing the input-surface nature now means you'll remember to write the grammar tests when you get there, not after the close has already missed them.

## Retrieving the epic's `agent_context`

When the issue you're about to claim is a child of an epic, the epic may carry `agent_context` — governing constraints (preferred approaches, forbidden patterns, review requirements) that apply to everything underneath it. These constraints are the ones most likely to bite mid-work if you miss them up front.

**The catch**: `agent_context` inheritance is **opt-in per project**. Even when set on an epic, descendants don't surface it on `br show` or `br update --claim` unless one of the following is true:

```yaml
# In .beads/config.yaml
inherited_context:
  enabled: true
```

```bash
# Or per-invocation via env var (wins over config)
BR_INHERITED_CONTEXT=1 br update <id> --claim
```

If you can't tell whether the project has inheritance enabled, **assume it doesn't** and check explicitly:

```bash
# Find the parent epic
br show <id> --json | jq -r '.parent_id'

# Read the epic's agent_context directly
br show <parent-id> --json | jq -r '.agent_context'
```

If `agent_context` is non-null on the epic, treat its contents as constraints on your work, not advisory notes. The whole point of the field is "this is what you'd otherwise have to rediscover painfully" — preferred approaches, rejected alternatives, coverage floors, review requirements. Honor it.

For deep dives on the `agent_context` schema, what to put there, and how inheritance works when enabled, see `plan-to-beads/agent-context.md`. This skill's concern is only retrieval at claim time.

### When the chain is deeper than one level

`br`'s v1 inheritance emission (when enabled) surfaces at most two blocks: the **immediate parent** and the **root ancestor** (preferring the topmost epic). If your issue is `br-a.1.2` (grandchild of an epic), inheritance — when on — will show you the epic `br-a` and the immediate parent `br-a.1`. When checking manually:

```bash
# Walk the parent chain
br show <id> --json | jq -r '.parent_id'           # → br-a.1
br show <parent-id> --json | jq -r '.parent_id'    # → br-a
br show <grandparent-id> --json | jq -r '.agent_context'
```

In practice, most epics live one or two levels above their children. If you find yourself walking three or more levels, the issue tree is probably over-nested — but that's a `plan-to-beads` concern, not an execution-time one. For now, just walk the chain.

## The `--claim` mechanics

When you're satisfied the issue is right, claim it:

```bash
br update <id> --claim
```

What this does atomically:

- Sets `status` to `in_progress`.
- Sets `assignee` to the current actor (resolved from `--actor`, `BR_AGENT_NAME` env, or default).
- Records the transition in `br audit log <id>`.
- Auto-flushes to JSONL.

What it validates:

- **Blocking dependencies**: if anything in `br dep tree <id>` of type `blocks` is still open, `--claim` fails with exit code 5 (`Dependency`). `--force` bypasses this check; use only when you've decided the blocker is actually irrelevant to your current work.
- **Existing assignment**: if the issue is already claimed by someone else (any actor that isn't you), `--claim` fails. `--force` overrides, but doing so without first leaving an audit comment is bad form. See [recovery.md](recovery.md) for the "same-agent stuck claim" pattern.

### Setting the actor explicitly

If multiple agents share a workspace, or if you want the audit log to show your identity rather than the default, set `--actor`:

```bash
br update <id> --claim --actor my-agent-name
```

Or set it once via env:

```bash
export BR_AGENT_NAME=my-agent-name
br update <id> --claim
```

Precedence is `--actor` > `BR_AGENT_NAME` > default. The actor lands in the audit log on every mutation, so it's worth getting right at session start rather than fixing later.

### What `--claim` does **not** do

- It does **not** start a timer. Estimates and elapsed-time tracking are outside `br`'s scope.
- It does **not** lock the issue against concurrent edits. Two sessions with the same actor running `br update <id>` in parallel can both succeed; the audit log will show the sequence. In a solo-agent workflow this is rarely a problem; just be aware.
- It does **not** validate the issue's content. A claim succeeds regardless of whether the description is vague, the acceptance criteria are missing, or the design field has been empty for a month. The read-before-claim discipline above is your only defense.

## Status transitions other than `--claim`

`--claim` is shorthand for "set status to in_progress and assign to me." Sometimes you want one without the other.

```bash
# Move to in_progress without changing assignee
br update <id> --status in_progress

# Take ownership without starting work yet (rare; usually claim is better)
br update <id> --assignee my-agent-name

# Move back to open (rare; usually --close or stays in_progress)
br update <id> --status open

# Mark blocked (when a real blocker emerges; prefer dep-add with --type blocks)
br update <id> --status blocked
```

Two notes:

1. **`--status in_progress` runs the same blocked-dependency check as `--claim`.** If the issue is blocked, both fail without `--force`. This is intentional — you shouldn't be moving an issue to in-progress when something it depends on is still open.

2. **`--status blocked` is usually the wrong shape.** When work surfaces a real blocker, the better move is to *file a new issue for the blocker* and add a `blocks` dependency. That way the blocker is itself a tracked unit of work, not just a flag. See [discovery.md](discovery.md) under "new bead needed."

## A worked session start

A concrete example of the orienting sequence, with the kinds of decision points that come up:

```bash
$ br sync --status
JSONL ahead (3 records)

$ br sync --import-only
Imported 3 records.

$ br list --status in_progress --json | jq -r '.issues[] | "\(.id): \(.title) (assignee: \(.assignee))"'
br-42: Refactor session middleware (assignee: me)

$ br show br-42 --json | jq -r '.notes'
2026-05-23 session: started extracting the session interface into
internal/auth/session.go. COMPLETED: interface defined, two callers
updated. NEXT: update remaining callers (auth handlers, admin tools).
NO BLOCKERS.
```

OK — past-me claimed `br-42`, made progress, didn't close. The notes are clear about where to pick up. I'll resume `br-42` rather than picking something new from `br ready`. No need to re-claim (already mine), no need to read the epic's `agent_context` again unless I've forgotten it (worth checking anyway):

```bash
$ br show br-42 --json | jq -r '.parent_id'
br-auth-overhaul

$ br show br-auth-overhaul --json | jq -r '.agent_context'
{
  "preferred_approach": "interface-first refactor; concrete types behind interfaces",
  "forbidden": ["direct sqlite imports outside storage package"],
  "review_required": false,
  "test_baseline": "must maintain >90% coverage in auth/"
}
```

Constraints noted. Ready to work. The next stop in the loop is [notes-discipline.md](notes-discipline.md) — what to write to `notes` as I make further progress, and when.

## Checklist

Before you write a single line of code:

1. **Ran `br sync --status` and `--import-only` if needed.**
2. **Checked `br list --status in_progress` for inherited claims.**
3. **Read `br show <id>` — description, acceptance, design, notes.**
4. **Noticed whether the issue changes a user-facing input surface** (flag, param, key, schema). If so, [input-surface-testing.md](input-surface-testing.md) applies at implementation and close.
5. **Walked the parent chain to find any `agent_context` on the epic** (since inheritance may not be enabled).
6. **Confirmed the issue is still the right shape** — if not, jumped to [discovery.md](discovery.md) before claiming.
7. **Claimed atomically with `br update <id> --claim`** (or resumed an existing claim of yours).
8. **Set `--actor` or `BR_AGENT_NAME`** if the default isn't what you want in the audit log.
