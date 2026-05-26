---
name: beads-to-done
description: Execute work tracked in beads_rust (`br`) issues — claim a ready issue, do the work, record progress, capture discovered work without derailing, close cleanly, and keep the JSONL/DB pair in sync. Solo-agent and cross-session-by-same-agent only. Use when issues already exist in `.beads/` and an agent needs to work them across one or many sessions. NOT for creating issues from a planning document (use `plan-to-beads` for that) and NOT for multi-agent coordination.
---

# Beads to Done

This skill covers the **execution phase**: an agent sits down to a populated `.beads/` and needs to do the work — claim, implement, record, close — across one or many sessions of the same agent.

It is the partner to `plan-to-beads`: planning produces beads; this skill carries them to done.

## When this skill applies

- An agent is starting a session against a repo with `.beads/` already populated.
- An agent is resuming work the same agent (possibly past-self, possibly different session) left mid-flight.
- An agent is closing out a finished issue.
- A discovery during work warrants capturing a new issue without derailing the current task, or reveals that the current issue is wrong.
- The JSONL and the DB are out of sync after a `git pull` (or for any other reason).

This skill does **not** cover:

- **Creating issues from a planning document.** That's `plan-to-beads`'s job. When discovery during execution warrants a new issue, this skill points back at `plan-to-beads` for the field-content discipline.
- **Multi-agent coordination / swarm features.** This skill is solo-agent and cross-session-by-same-agent only. `br coordination status`, exclusive claims, scheduler stale-claim policy, reclaim across competing agents, Agent Mail integration — out of scope. If a project uses swarms, this skill is the wrong skill.
- **`br` setup, install, repo init.** Assumes a working `br` and a populated `.beads/`.

## The core principle

**The issue is the unit of work. The JSONL is the audit trail. The DB is the working copy.**

Every mutation — claim, status change, notes update, close — flows through `br`, never through hand-editing `.beads/issues.jsonl`. The JSONL exists so a future session (or a teammate, or future-you) can reconstruct exactly what happened and resume. If the JSONL drifts out of sync with the DB, that's a problem the [recovery](recovery.md) flow fixes — not a problem to ignore.

The single most useful mental model: **the issue's fields are the only source of truth for the work**. After a session restart or context compaction, the agent will see only what `br show <id>` returns. Anything kept in conversation context but not written to the issue is gone. Resumability is a property of how the agent writes, not of how clever the next session is.

## The execution loop at a glance

```bash
# 1. Session start: orient
br sync --status                              # JSONL vs DB?
br sync --import-only                         # if JSONL is ahead
br list --status in_progress --json           # anything already claimed?
br ready --json                               # what's unblocked?

# 2. Pick and claim
br show <id>                                  # read everything first
br update <id> --claim                        # atomically take it

# 3. Work the loop
# ... implement ...
br update <id> --notes "<read-then-write snapshot>"   # at milestones

# 4. Close
br close <id> --reason "<what + commit ref>" --suggest-next --json

# 5. Sync and ship
br sync --flush-only                          # idempotent; auto-flush usually has this covered
git add .beads/
git commit -m "Close <id>: short summary"
git push
```

Five phases, in order. Skipping any of them tends to surface as a problem one session later, when a future agent (or future-you) can't tell what happened.

## Session mode: single-issue vs queue

The loop above closes one issue. Before starting, decide whether the user wants you to **stop after this one** or **keep going until the queue is empty (or some scope is exhausted)**. Getting this wrong is the most common way to under- or over-deliver.

### Single-issue mode (default)

The user named a specific issue or a small, finite set. Work the loop once per issue, then stop and report back. Typical signals:

- "Work `br-42`."
- "Finish the auth refactor bead."
- "Close out the issue I claimed yesterday."

Run the loop, close, report. Don't reach for `br ready` after closing.

### Queue mode

The user wants the queue drained (possibly within a scope). Typical signals:

- "Work through the auth epic."
- "Drain the queue."
- "Keep going until ready is empty."
- "Finish everything that's ready."
- "Get as far as you can this session."

In queue mode, after every close:

```bash
br ready --json
```

If non-empty:

- **Prefer newly-unblocked siblings under the same epic** before picking unrelated work. Closing a leaf often makes its parent epic closeable (see "Epic close-out" in [closing-and-sync.md](closing-and-sync.md)) or unblocks sibling leaves; finishing the epic before context-switching is usually higher-value than starting a new branch.
- Pick the top item per the normal `br ready` priority sort.
- Run the loop again.

If empty:

- Apply the **queue-empty definition of done** below.

### When the scope is in between

Often the user names a scope that's bigger than one issue but smaller than the whole queue — "the auth epic," "all the P0s," "the things blocking the release." Treat it as queue mode bounded by the scope:

- After each close, check `br ready` *filtered by the scope* (`--priority 0`, an epic's children via `br list --parent <epic-id> --status ready`, etc.).
- Stop when *the scoped subset* is empty, not when the whole queue is.

Report what's left outside the scope at the end so the user can decide what's next.

### Queue-empty definition of done

When operating in queue mode and `br ready` (or its scoped subset) returns empty, verify the workspace is actually clean before reporting "done":

```bash
br status                          # 0 open/in_progress/blocked/ready in scope
br sync --status                   # "in sync"
git status --short --branch        # clean working tree, branch up to date
git log -3 --oneline | cat         # confirm the latest closes are pushed
```

The four signals together mean: nothing left to do in scope, JSONL and DB agree, working tree is clean, and the work is shipped. Report this state to the user explicitly — don't just say "done." A useful pattern:

> Queue drained. 7 issues closed (`br-42`..`br-48`), 3 epics closed (`auth-1`, `auth-2`, `auth`). `br ready` empty in scope. Sync clean, branch pushed to origin/main.

If any of the four signals is off, fix it before reporting done. A queue that's "empty" but with an unpushed commit isn't actually done.

### When to bail out of queue mode early

Queue mode does not mean "work until you crash." Stop and surface to the user when:

- A [discovery branch 3](discovery.md) hit ("stop and ask").
- A blocker filed during work blocks the rest of the scoped queue.
- Context is heavy enough that the next close risks losing notes — finish the current issue's notes/close honestly, then report and let the user start a fresh session.
- An unfamiliar safety guard fires during sync. Don't `--force` through it in batch mode; surface to the user.

In each case, leave the workspace in a clean, resumable state (notes current on whatever's `in_progress`, no half-finished mutations) before reporting.

## Dispatch: which detail file applies

This `SKILL.md` is the loop at a glance. Detail lives in five supporting files:

| If you're… | Read |
|---|---|
| Starting a session, picking ready work, claiming an issue, retrieving an epic's `agent_context`, sanity-checking an issue before starting | [claim-and-work.md](claim-and-work.md) |
| Updating `notes`, deciding between `--notes` and `br comments add`, structuring a resumable progress snapshot | [notes-discipline.md](notes-discipline.md) |
| Encountering an unexpected behavior, a missing prerequisite, scope creep, or a wrong-issue situation mid-work — deciding whether to spawn a new bead, update the current one, or stop | [discovery.md](discovery.md) |
| Closing an issue (incl. epic close-out and the commit-ordering convention), writing a traceable `--reason`, syncing JSONL/DB, the git half of the close | [closing-and-sync.md](closing-and-sync.md) |
| JSONL/DB divergence, conflict markers in JSONL, a stuck `in_progress` claim from a crashed session, `br doctor` | [recovery.md](recovery.md) |

## Cross-references to `plan-to-beads`

This skill assumes issues already exist. When work reveals a new issue that needs creating, **don't redocument the creation grammar here** — defer to `plan-to-beads`:

- Field-content discipline (what goes in `description` vs `design` vs `acceptance` vs `notes`): `plan-to-beads/field-semantics.md`.
- ID shape, parent/child IDs, dependency types (`blocks`, `discovered-from`, `related`, `external`): `plan-to-beads/structure.md`.
- Bulk creation grammar: `plan-to-beads/bulk-import.md`.
- Setting governing context on an epic (`agent_context`): `plan-to-beads/agent-context.md`.

This skill's [discovery.md](discovery.md) covers *when* a discovery merits a new issue and how to capture it without derailing — but the *how to construct* the new issue is `plan-to-beads`'s territory.

## A note on `br` quirks worth carrying into the loop

These bite agents who don't know about them. Each is covered in detail in the file where it actually matters:

- **`--notes` overwrites.** Read existing notes before writing or you'll erase history. See [notes-discipline.md](notes-discipline.md).
- **Auto-flush is on by default**, so mutations land in JSONL as they happen — but `br sync --flush-only` is a cheap, idempotent final check before staging. See [closing-and-sync.md](closing-and-sync.md).
- **`br` never runs git.** Staging, committing, pushing are always explicit. See [closing-and-sync.md](closing-and-sync.md).
- **`agent_context` inheritance is opt-in per project.** Even when set on an epic, descendants don't surface it unless `inherited_context.enabled: true` (or `BR_INHERITED_CONTEXT=1`). Check explicitly with `br show <epic-id> --json` rather than assuming. See [claim-and-work.md](claim-and-work.md).
- **Bulk operations are not transactional.** `br create -f` and other bulk mutations can partially succeed and emit warnings in the same run. Inspect `--json` output rather than trusting exit code alone. (This skill rarely uses bulk operations, but worth knowing.)
- **`br lint` is not a generic validator.** Some rule sets check for headings inside `description` that conflict with separated-fields philosophy. Run `br lint` only if the project's rules align with separated fields.
- **Live behavior > memorized flags.** When in doubt about a flag or output shape, `br capabilities --command <name> --format json` and `br <subcommand> --help` are the live source of truth.

## Quick checklist for any execution session

1. **Did you confirm session mode** — single-issue or queue? See "Session mode" above.
2. **Did you `sync --status` at the top?** Cheap, prevents stale-DB surprises.
3. **Did you check `list --status in_progress` before claiming new work?** A stuck claim from a previous session is the first thing to handle.
4. **Did you read `br show <id>` *and* the epic's `agent_context` (if any) before starting?** Mid-work surprises are usually constraints that were on the parent epic.
5. **Are you updating `notes` at milestones, not just at the end?** Context compaction is not negotiable; the only record of what happened is what you wrote.
6. **When the work surprised you, did you correctly route the surprise** — update the current issue, spawn a new one, or stop and ask? See [discovery.md](discovery.md).
7. **Does the close `--reason` include both a summary and a commit ref?** "Done" or "Implemented" leaves the next agent with nothing.
8. **In queue mode, did you re-check `br ready` after every close** and apply the queue-empty definition of done before reporting?
9. **Did `git push` succeed?** Work is not done until the JSONL is on the remote.

## Supporting files

- **[claim-and-work.md](claim-and-work.md)** — Session start, finding ready work, the `--claim` mechanics, status transitions, retrieving an epic's `agent_context`, the read-before-claim check that catches a wrong issue before you start.
- **[notes-discipline.md](notes-discipline.md)** — `--notes` overwrite semantics, read-before-write pattern, `--notes` vs `br comments add`, structuring a resumable progress snapshot.
- **[discovery.md](discovery.md)** — Heuristics and worked examples for the three discovery branches: new bead needed, current bead is wrong, scope creep. Wiring `discovered-from` dependencies. When to stop and ask the user.
- **[closing-and-sync.md](closing-and-sync.md)** — Acceptance walkthrough, notes cleanup, traceable close reasons, the implementation-commit / close / tracker-commit ordering, epic close-out (no-code closes), `--suggest-next`, the JSONL/DB model, sync commands.
- **[recovery.md](recovery.md)** — `sync --status` decision tree, three-way merge, force modes, JSONL conflict markers, history restore, `br doctor`, and the same-agent stuck-claim case.
