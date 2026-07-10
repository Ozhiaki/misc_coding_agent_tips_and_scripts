# Notes Discipline

The `notes` field is the load-bearing mechanism that makes work resumable across sessions. The discipline is small but unforgiving: `--notes` **overwrites**. Forgetting that erases history. This file is about getting the mechanics and the structure right.

## The core mental model

After a session restart or context compaction, a future agent (often future-you) will see only what `br show <id>` returns. They will not see your conversation history. They will not see your scratchpad. They will see, at most:

- `description` — what the problem is. Immutable.
- `acceptance` — how to verify it's done. Stable.
- `design` — how you're solving it. Possibly evolved.
- `notes` — where you left off. Possibly the only signal of progress.

If `notes` is empty or stale, the future agent has to reconstruct your work from the diff. That works when the diff is small and self-explanatory. It fails the moment the work is incomplete, has open questions, or involved a non-obvious decision.

**Treat `notes` as the message-in-a-bottle you're writing to your future self after they've lost everything.** Then write accordingly.

## Public-safe writing rule

If `.beads/issues.jsonl` may be committed, pushed, shared, or public, notes and comments are publishable project history. They are not a private scratchpad, chat transcript, or agent session log.

Record the state needed to resume:

- What is completed.
- What is in progress.
- What is next.
- What is blocked.
- Which durable decision was made and why, when the rationale is technical and public-safe.

Do not record private provenance or process residue:

- Raw prompts, conversation snippets, "user said" attribution, session IDs, compaction summaries, subagent/tool/mail routing.
- Local absolute paths, machine/user names, temp directories, private repo/module paths.
- Secrets, tokens, credentials, auth headers, private registry settings, or credential validation details.
- Legal, licensing, trademark, customer, outreach, repo-visibility, publication, or launch-gate deliberations.
- Private tag/branch validation, private CI, internal release rehearsals, or old organization/module/package names not intended for the public record.

Rewrite private context into durable project facts. For example: "Public install path is `go install github.com/org/project/cmd/tool@latest`" is useful; "Dave said after private launch discussion to use..." is not.

## The footgun: `--notes` overwrites

`br update <id> --notes "..."` **replaces** the existing notes content. There is no `--append-notes` flag. If you run:

```bash
br update br-42 --notes "Started work on auth refactor."
```

…and then later run:

```bash
br update br-42 --notes "Tests passing."
```

…the first note is gone. Forever. (Well — not forever. `br audit log` retains the history, and `br history list` retains backup snapshots of the JSONL. But for any normal `br show`, that earlier content is invisible.)

This is by design — `notes` is a *current-state snapshot*, not a chronological log. The append-only role is filled by `br comments add`; see below.

## The read-before-write pattern

The universal pattern for updating notes:

```bash
# 1. Read existing notes
br show br-42 --json | jq -r '.notes'

# 2. Compose new content that includes the relevant past content
# (You can drop content that's no longer relevant — that's the point of overwrite semantics.)

# 3. Write combined content
br update br-42 --notes "Previous content (trimmed)...
---
2026-05-25 session: <new update>"
```

Two practical refinements:

### Use a date separator

Date-stamp each session's contribution and separate with `---`. This makes the chronology legible and makes it obvious where to trim when notes get long:

```
2026-05-23: Initial pass. Interface defined. Two callers updated.
NEXT: update remaining auth handlers.
---
2026-05-24: Updated auth handlers. Admin tools next.
NEXT: admin/users.go, admin/sessions.go.
---
2026-05-25: All callers updated. Tests passing locally. Coverage 92%.
NEXT: run integration suite, then close.
```

### Trim aggressively when the snapshot grows stale

`notes` is not a journal. When earlier session content is no longer load-bearing for the *current* state, drop it. The audit log preserves history if you ever need it back. The goal is "a future agent can resume in two minutes," not "a future agent can reconstruct every step."

A reasonable heuristic: if a past session's content describes work you've already obsoleted, replaced, or that's covered by `design` updates, trim it.

## A structure that works

The handoff template that survives compaction well:

```
COMPLETED:
- <thing 1>
- <thing 2>

IN_PROGRESS:
- <thing currently in flight, with file/function targets if possible>

NEXT:
- <the very next concrete action>
- <the action after that>

BLOCKERS:
- <anything stopping progress; usually empty>

KEY DECISIONS:
- <decision and why, if non-obvious>
```

You don't have to use this exact shape, but the four buckets it implies — done, in flight, next, blocked — are the ones a future agent actually needs. Free-form prose tends to bury the actionable bits.

For very short work where this structure is overkill, a single sentence is fine:

```
2026-05-25: All callers updated, tests passing locally. NEXT: run integration suite, close.
```

The structure scales down. What it shouldn't scale to is *nothing* — empty notes on an `in_progress` issue is the worst-case bequest to a future session.

## `--notes` vs `br comments add`

`br` has two related but distinct facilities for session-by-session writing:

| | `--notes` (overwrite) | `br comments add` (append-only) |
|---|---|---|
| **Semantics** | Replaces existing | Appends a new entry |
| **Purpose** | "Where am I right now?" | "What did I try / decide?" |
| **Read order** | Single field, always current | List, newest-first or oldest-first |
| **Survives compaction** | Yes (summarized) | Yes (summarized) |
| **Lost on accident** | Yes (overwrite footgun) | No (append-only) |
| **Best for** | Handoff state, NEXT step | Decisions, tried-and-abandoned approaches, rationale |

Practical division of labor:

- **Use `--notes`** for the canonical "where I am" snapshot. The thing a future agent reads first.
- **Use `br comments add`** for things that are valuable to preserve but not load-bearing for resumption: "tried X, didn't work because Y," "considered Z, rejected because W," "decision: prefer A over B because it preserves backward compatibility."

Commands:

```bash
br comments add br-42 "Tried using sync.RWMutex; readers block too long under load. Switching to sync.Mutex."
br comments list br-42                  # newest-first, default
br comments list br-42 --format json    # machine-readable
br comments list br-42 --oldest-first   # if you want chronological
```

A useful pattern: when `notes` says "tried X, switched to Y," `comments` should have the *why*. Notes carries forward, comments carries depth.

## When to update notes

The trigger should not be "at session end." That's too late — a session might end abruptly (compaction, crash, the user closing the laptop). Update at:

1. **Milestones during work.** Every time you've completed a non-trivial sub-step (a feature compiles, tests pass for one component, a tricky refactor lands). Aim for at most ~30 minutes of work between updates.
2. **Before any risky operation.** About to run a big refactor? Update notes first. About to attempt something you might have to abandon? Update notes first. The notes you wrote 30 seconds ago are worth more than the notes you intended to write.
3. **When context is getting heavy.** If you notice your conversation is getting long, update notes now. Compaction can happen at unpredictable times; you don't get a warning.
4. **At explicit handoff points.** Before saying "done," before stepping away, before switching to a different issue.

What *not* to use as a trigger: "I'll update at the end." End-of-session updates lose to mid-session crashes 100% of the time.

## What to keep out of notes

Notes is for *session-level state*. Things that don't belong:

- **The problem statement.** That's in `description`. Don't restate it in notes.
- **The implementation approach.** That's in `design`. If your approach evolved, update `design` — don't write "actually we ended up doing X" in notes.
- **The acceptance criteria.** Those are in `acceptance`. If acceptance changed, that's a sign the issue's shape changed; see [discovery.md](discovery.md).
- **Permanent decisions about the codebase.** Those belong in code comments, an ADR, or `design`. Notes is ephemeral by design.
- **Conversational fragments.** "User said to do it this way" → put the decision itself in notes, not the meta-comment about who said it.
- **Private process details.** Session names, prompts, tool/mail routing, local paths, credential checks, and launch/legal/customer deliberations belong outside committed tracker history.

Said differently: if the thing you're about to write would still be relevant six months from now, it's not notes. It's `design`, or `description` (rare — usually immutable), or a code comment, or its own issue.

## A worked example

You're working on `br-42` ("Refactor session middleware"). Session 1, you make progress and write:

```bash
br update br-42 --notes "2026-05-23: Initial pass. Defined Session interface in internal/auth/session.go. Updated two callers (login.go, logout.go). NEXT: update remaining callers — admin handlers and the CLI tool."
```

Session 2, two days later. You sit down, read existing notes (good — you remember the read-before-write rule), do more work, then update:

```bash
# Read existing
$ br show br-42 --json | jq -r '.notes'
2026-05-23: Initial pass. Defined Session interface in internal/auth/session.go.
Updated two callers (login.go, logout.go). NEXT: update remaining callers — admin
handlers and the CLI tool.

# Write combined
br update br-42 --notes "2026-05-23: Initial pass. Defined Session interface in internal/auth/session.go. Updated login.go, logout.go.
---
2026-05-24:
COMPLETED: admin/users.go, admin/sessions.go updated.
IN_PROGRESS: cli/auth.go — partial; AuthCommand still uses old API.
NEXT: finish cli/auth.go, then run integration tests.
BLOCKERS: none.
KEY DECISIONS: kept the old Session struct as SessionLegacy temporarily so the diff is reviewable; will remove in a follow-up issue."
```

The earlier content is preserved (trimmed lightly — dropped the redundant "interface in ... session.go"). The new session adds structured progress. A future session can pick up at "finish cli/auth.go" with no other context.

Note also the `KEY DECISIONS` line. The detail there — keeping `SessionLegacy` temporarily — is the kind of thing that's painful to rediscover. Some of it could also go to `br comments add`:

```bash
br comments add br-42 "Decision: kept old Session struct as SessionLegacy during refactor. Will remove in follow-up issue once all callers migrate. Tried removing in one pass; too much surface to review."
```

Now the *what* is in notes (NEXT-relevant), and the *why* is in comments (audit-relevant).

## Mistakes the discipline prevents

This is what bad notes hygiene looks like, and what each mistake costs:

| Mistake | Cost |
|---|---|
| Forgot to read before writing → erased prior notes | Future agent has no idea what was done; reconstructs from diff. |
| Updated only at session end → crash before update | Future agent inherits an `in_progress` issue with stale notes; misroutes effort. |
| Wrote unstructured prose | Future agent can't quickly find "NEXT"; spends time parsing. |
| Stuffed `design`-level content into notes | `design` field stays stale; next agent looks in the wrong place. |
| Empty notes on `in_progress` | The worst case. Future agent has nothing. |
| Used notes as a chat log ("user said...") | Information is unsearchable; meta-noise crowds out actionable state. |

None of these are unrecoverable. All of them are avoidable by following the read-before-write pattern, updating at milestones, and keeping notes structured and current-state-focused.

## Checklist

- **Before updating: `br show <id> --json | jq -r '.notes'`.** Always.
- **After updating: skim the result with `br show <id>`** to confirm the new content reads cleanly.
- **Structure**: COMPLETED / IN_PROGRESS / NEXT / BLOCKERS / KEY DECISIONS. Or trim for short work.
- **Frequency**: at milestones (every ~30 min of progress), before risky ops, when context is heavy, at handoff.
- **`--notes` is current-state**; **`br comments add` is decision-trail**. Use both, for different things.
- **Don't put `design`/`description`/`acceptance` content in notes.** Different field, different mutability.
- **If `.beads/` is committed or shared, keep notes/comments publishable.** Preserve technical state; omit private provenance and process residue.
