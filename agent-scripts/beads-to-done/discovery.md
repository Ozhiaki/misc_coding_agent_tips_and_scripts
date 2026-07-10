# Discovery

You're working an issue and something unexpected shows up. This file is the decision aid for what to do about it: spawn a new bead, update the current one, or stop and ask the user.

The discipline matters because the alternatives — silently expanding the current issue, or burying the surprise in code and moving on — both look easier in the moment and both cost more later. The current issue becomes a kitchen sink; the surprise gets forgotten until it bites a future session.

## Public-safe discovery capture

Discovery often happens in the messiest part of a session. If `.beads/issues.jsonl` may be committed, pushed, shared, or public, slow down before writing notes, comments, or new issues. Capture the technical fact and durable decision; do not capture private provenance.

Good durable content:

- "Acceptance was updated because the prior criterion was unverifiable against the API schema."
- "Blocked pending product decision between per-IP and per-account rate limiting."
- "Filed br-99 for deprecated API migration; not blocking current work."

Do not write:

- Raw conversation fragments, "user said" attribution, session IDs, subagent/tool/mail routing, or compaction summaries.
- Local absolute paths, private repo/module names, credentials, private registry settings, auth validation details.
- Legal, licensing, trademark, customer, outreach, repo-visibility, publication, or launch-gate deliberations.
- Private tag/branch validation, private CI, internal release rehearsals, or old organization/module/package names not intended for the public record.

## The three branches

Almost every mid-work surprise resolves into one of three branches:

1. **New bead needed.** The surprise is real, valid work, *separate from* the current issue. File a new bead, wire a `discovered-from` dependency, and **keep working the current issue**.
2. **Current bead is wrong.** The surprise reveals that the current issue's description, design, or acceptance criteria don't match reality. Fix the current issue's fields (or stop and ask the user, depending on severity).
3. **Stop and ask the user.** Some surprises are above your pay grade — they imply a scope change, a real architectural decision, or a contradiction in the plan you can't resolve. Stop, write notes capturing the state, and surface the question.

These aren't mutually exclusive — a surprise can require updating the current issue *and* filing a new bead — but the first question is always "which of these is this?"

## Heuristic: which branch are you in?

The fastest test that's right most of the time:

> Could the current issue be closed exactly as written if this surprise were also resolved separately?

- **Yes** → branch 1 (new bead). The current issue is fine; the surprise is its own thing.
- **No, the current issue's *fields* would need to change** → branch 2 (current bead is wrong). Update fields in place.
- **No, the current issue's *whole shape* is wrong, or this implies a decision you can't make alone** → branch 3 (stop and ask).

Two finer-grained rules of thumb:

- **If the surprise is a problem you didn't know existed, but it doesn't change what "done" looks like for the current issue → branch 1.**
  - Example: while refactoring auth, you notice the logging module uses a deprecated API. Doesn't affect the auth refactor. File `br create "Migrate logging from deprecated API"`, add `discovered-from` dependency, keep going.
- **If the surprise changes what "done" looks like for the current issue → branch 2 or 3.**
  - Example: while implementing the acceptance criterion "rate-limit logins to 5/min," you discover the existing rate-limiter has no per-IP granularity, only global. The current acceptance criterion is now ambiguous: per-IP or global? Update `design` and `acceptance` if you can answer it, otherwise stop and ask.

## Branch 1: new bead needed

The most common branch and the most prone to "I'll just fix it while I'm here" temptation. Resist. Mixing two pieces of work into one issue makes both worse:

- The close `--reason` won't honestly describe what changed.
- The diff is bigger and harder to review.
- Future search for the discovered issue finds nothing (because it was never filed).
- The audit log loses the relationship.

### How to capture without derailing

The capture should take less than 60 seconds and should not pull you out of the current work mentally. Two patterns:

**Pattern A: file immediately, full content.** Best when the surprise is small and self-contained:

```bash
br create "Migrate logging from deprecated log/syslog to log/slog" \
  -t task -p 2 \
  -d "While refactoring auth middleware, noticed internal/logging/syslog.go uses log/syslog (deprecated as of Go 1.21). Affects all log call sites in internal/. Not blocking auth refactor."

# Capture the new ID (e.g., br-99) and wire the relationship
br dep add br-99 br-42 --type discovered-from
```

**Pattern B: `br q` placeholder, then come back.** Best when the surprise is significant enough that you want to file it properly but you don't want to break flow right now:

```bash
br q "Logging module uses deprecated log/syslog — see br-42 context"
# Returns an ID, e.g., br-100. Now back to work.
```

Then, at the next milestone (next time you're updating notes anyway), come back and fill out the placeholder:

```bash
br update br-100 \
  -d "While refactoring auth (br-42), noticed internal/logging/syslog.go uses log/syslog (deprecated as of Go 1.21). All internal/ log sites affected."
br dep add br-100 br-42 --type discovered-from
```

Field-content discipline for the new issue is `plan-to-beads/field-semantics.md`'s territory — `description` is the *why*, `design` and `acceptance` come at create time or via follow-up `br update`, no "as discussed" / "per the plan" language.

### Wiring the `discovered-from` dependency

The `discovered-from` edge type is what makes the relationship traceable. It says: "issue B was discovered while working on issue A." It does **not** make B block A; you can still close A independently. It's a paper trail, not a gate.

```bash
br dep add <new-id> <original-id> --type discovered-from
```

Order matters: `<new-id>` first, then `<original-id>`. Read it as "new-id was discovered-from original-id." Get this backwards and `br dep tree` will show the relationship in the wrong direction, which mostly works fine until somebody (you, six months from now) is trying to figure out which issue caused which.

If the discovered work *does* block the current issue — you found out you can't finish A without finishing B first — that's a different relationship: `--type blocks`:

```bash
br dep add <new-id> <original-id> --type blocks
```

And then the current issue's status should be reconsidered. See "When a discovery blocks the current work" below.

### Don't mention the new bead inline in code

It's tempting to leave a `// TODO: see br-99` in the code where you noticed the issue. Don't, unless the project specifically wants that. The audit trail is in `br`, not scattered through the codebase. The `discovered-from` edge does the linking job. Inline `br-` references in code rot when issue IDs change (renumbering during a workspace merge, for instance) and clutter the diff.

A note in the *current* issue's `--notes` is fine, and often valuable:

```bash
br update br-42 --notes "<previous content>
---
2026-05-25: While updating auth/middleware.go, noticed deprecated log/syslog usage in internal/logging. Filed br-99 with discovered-from edge. Not blocking current work."
```

That keeps the breadcrumb in the issue tracker where it belongs.

## Branch 2: current bead is wrong

This is the more uncomfortable branch and the one agents most often mishandle by just plowing through. Three sub-cases by what's wrong:

### 2a. Acceptance criteria don't match reality

You read `acceptance` at claim time, it seemed reasonable, but mid-work you realize one of the criteria is unverifiable, contradicted by another, or describes a state the code is already in.

Two sub-sub-cases:

- **The criterion is wrong, but you can clearly tell what it *should* say** (e.g., a typo, an obvious oversight): update it.

  ```bash
  br update <id> --acceptance-criteria "<corrected criteria, preserving everything not wrong>"
  ```

  Add a comment recording why:

  ```bash
  br comments add <id> "Updated acceptance: original criterion 'returns 200 on success' was ambiguous; changed to 'returns 200 with JSON body matching schema X' to match the rest of the API's convention."
  ```

- **The criterion is wrong, and you're not sure what it should say.** Don't guess. Branch 3 (stop and ask).

### 2b. Design is impossible or stale

You read `design` at claim time, started implementing, and ran into a wall: the approach can't work, an API has changed, a library doesn't exist, the prerequisite was wrong.

Update `design` to reflect what you now know:

```bash
br update <id> --design "<new approach reflecting reality>"
```

Then `br comments add` with the rationale:

```bash
br comments add <id> "Design updated: foo.NewClient(...) was removed in foo v0.18. Switched to foo.MustNewClient with explicit config, matching upstream public docs."
```

This is also a good moment to update `notes` to reflect that the approach changed:

```bash
br update <id> --notes "<previous content>
---
2026-05-25: Design changed — see updated design field. Original approach blocked by foo v0.18 breaking change. NEXT: continue with revised approach."
```

The point is to leave a future session a coherent state: `design` says what we're doing, `comments` says why we changed, `notes` says where we are.

### 2c. Description is wrong

This is the most serious version of "current bead is wrong" and almost always branch 3 (stop and ask). The `description` field is the *problem statement* — what the issue exists for. If it's wrong, the issue itself is wrong, and any work you do on it will be solving the wrong problem.

Symptoms:

- The description says "the deploy script fails on macOS" but the deploy script actually works on macOS and fails on a specific CI image only.
- The description says "users can't log in" but the actual problem is "users with 2FA enabled can't log in via SSO."
- The description describes a problem that's already been resolved by an unrelated commit.

In any of these cases, the right move is to stop, update `notes` with what you've found, and ask the user whether to:

- Rewrite the description (rare — descriptions are supposed to be stable).
- Close the issue as obsolete (`br close <id> --reason "Already resolved by commit a1b2c3d"`).
- Split into a more accurate issue and close this one.

## Branch 3: stop and ask

When the surprise crosses a line that's not yours to cross alone, stop. The lines worth recognizing:

- **The surprise reveals a real scope change** ("this isn't a 1-hour fix, it's a 1-week refactor").
- **The surprise reveals a decision the user hasn't made** ("the spec is ambiguous between behavior A and behavior B").
- **The surprise contradicts something else you thought was settled** (the epic's `agent_context` says "use X" but X is now impossible; do you switch, or escalate?).
- **The surprise reveals a possible security or data-integrity concern** that's not in scope but that the user should know about now.

What to do:

1. **Stop coding.** Don't keep working "while waiting" — that's how scope creeps.
2. **Update `notes`** with where you are and what you found, written as a public-safe project state summary:

    ```bash
    br update <id> --notes "<previous content>
    ---
    2026-05-25 STOPPED: <one-line public-safe summary of the surprise and why it requires user input>.
    State at stop: <what's done, what's not>.
    Question: <the specific public-safe question the user needs to answer>."
    ```

3. **Surface the question to the user.** Concisely. With enough context that they can answer without re-reading the issue.

You don't need to close or release the claim — the issue stays `in_progress`, attributed to you, in a known-stopped state. When the user answers, you resume.

A `br comments add` entry is also worth dropping here, because the audit trail benefits:

```bash
br comments add <id> "Stopped mid-work pending user input. Notes field has the question. Will resume when answered."
```

## Worked examples

Three concrete scenarios, one per branch.

### Scenario 1: new bead (branch 1)

You're working `br-42` ("Refactor session middleware"). Mid-refactor, you notice that the existing logout handler doesn't invalidate the session token on the server side — it just clears the client cookie. That's a real bug, but it has nothing to do with the refactor.

```bash
# Capture quickly with br q
$ br q "Logout doesn't invalidate session server-side"
Created br-99

# Wire the relationship
$ br dep add br-99 br-42 --type discovered-from

# Brief note in current issue
$ br show br-42 --json | jq -r '.notes' > /tmp/notes.txt
# (append the discovery, write back)

$ br update br-42 --notes "$(cat /tmp/notes.txt)
---
2026-05-25: Noticed logout handler doesn't invalidate server-side session. Filed br-99 with discovered-from. Not blocking; continuing refactor."

# Back to work
```

Later, at the next milestone, fill out `br-99` properly:

```bash
br update br-99 \
  -d "Logout handler in auth/logout.go clears the client-side cookie but does not delete the session from the server-side session store. A reused cookie (or one captured before logout) remains valid until natural expiry. Discovered while refactoring br-42."

br update br-99 \
  --design "Add session deletion call to logout handler. Need to confirm session store supports synchronous delete." \
  --acceptance-criteria "Logout handler deletes the session from store. Reused cookie after logout returns 401. Test added."
```

The new issue is properly formed, the relationship is recorded, the current work wasn't derailed.

### Scenario 2: current bead is wrong, you can fix it (branch 2)

You're working `br-43` ("Add Sensitive boolean to SecretObject"). The acceptance criteria say:

> SecretObject has Sensitive bool defaulting to true.
> NewSecretObject() sets it.
> JSON omits field when true (omitempty).
> Existing secrets without Sensitive field default to true on load.

Mid-work, you realize the last criterion is impossible as written — `omitempty` means the field is omitted when *true* (because true is not the zero value… wait, the zero value of `bool` is `false`, so `omitempty` would omit *false* values, not true ones). The criterion confuses itself.

You re-read and figure out the intent: when an old JSON record without the `Sensitive` field is decoded, the result should have `Sensitive: true`. That requires either a custom UnmarshalJSON or a constructor that backfills. The criterion's *intent* is clear; the *wording* is wrong.

```bash
# Update acceptance with corrected wording
br update br-43 --acceptance-criteria "SecretObject has Sensitive bool defaulting to true.
NewSecretObject() sets Sensitive: true.
JSON encoding includes the field with its actual value (no omitempty — would cause false to be ambiguously omitted).
When decoding JSON that has no Sensitive key (legacy records), the resulting SecretObject has Sensitive: true (via custom UnmarshalJSON or post-decode backfill)."

# Record why
br comments add br-43 "Updated acceptance: original wording conflicted with bool omitempty semantics. Rewrote criteria so legacy records without Sensitive default to true on load."

# Update notes to reflect state
br update br-43 --notes "<previous>
---
2026-05-25:
COMPLETED: Sensitive field added; NewSecretObject sets it.
IN_PROGRESS: implementing UnmarshalJSON for legacy backfill.
NEXT: tests.
KEY DECISIONS: Removed omitempty per acceptance update. See comments."
```

The issue's fields now cohere. The audit trail explains why the change was made. Future sessions won't be misled by the broken criterion.

### Scenario 3: stop and ask (branch 3)

You're working `br-44` ("Add rate-limit to login endpoint, 5 attempts per minute"). The acceptance says:

> Returns 429 after 5 failed logins in 60s.

Mid-implementation, you notice the existing rate limiter is per-process, not per-IP. So "5 attempts per minute" could mean:

- 5 attempts per minute *globally* (anyone, anywhere — a denial-of-service magnet).
- 5 attempts per IP (probably intended, but requires upgrading the limiter).
- 5 attempts per username (most defensive, but enables enumeration attacks).

The choice is a real security tradeoff. The epic's `agent_context` doesn't address it. You can't guess.

```bash
# Stop, capture, surface
br update br-44 --notes "<previous>
---
2026-05-25 STOPPED: Acceptance says '5 attempts per minute' but existing rate limiter is per-process, not per-IP.

Three viable interpretations:
1. Per-process global (current limiter; bad UX, DoS risk).
2. Per-IP (intended? requires upgrading limiter to per-key).
3. Per-username (defends one user but enables enumeration via timing).

State at stop: limiter scaffolding in place in auth/ratelimit.go; the per-key tracker is the open piece.

Question: which scope? Or some hybrid (e.g., per-IP AND per-username, with the stricter limit winning)?"

br comments add br-44 "Stopped pending user input on rate-limit scope. Three interpretations of 'per minute' — see notes."
```

Then surface the question to the user. You don't release the claim; the issue stays `in_progress` and attributed to you. When the user answers, you resume from the notes.

## Pitfalls to avoid

| Anti-pattern | What it looks like | The fix |
|---|---|---|
| **"I'll just fix it while I'm here"** | Two pieces of work in one diff; close `--reason` doesn't honestly describe both | File the second piece as a separate bead, wire `discovered-from`, work them independently |
| **Inline `TODO: br-XXX`** | Stale references scattered through code | Use the issue tracker for tracking; let `discovered-from` do the linking |
| **Silent acceptance-criteria mutation** | Field changed without explanation | Always pair `--acceptance-criteria` update with a `br comments add` recording why |
| **"The design says X but I'm doing Y, I'll update later"** | Stale `design` field; future session implements against the stale plan | Update `design` *as the approach changes*, not at the end |
| **Plowing through a stop-and-ask case** | Major decision made silently; user later disagrees | Recognize the line; stop; ask |
| **Capturing the discovery only in `notes`, never as an issue** | Future search finds nothing | If it's real work, it gets a bead |
| **Filing the discovery but never wiring `discovered-from`** | Audit trail loses the relationship | Always add the edge |

## When a discovery blocks the current work

Sometimes the discovery isn't parallel — it's a real blocker for the current issue. Example: working `br-42` (refactor session middleware), you discover the session store has a corruption bug (`br-99`) that makes the refactor impossible to test until fixed.

The right shape:

```bash
# File the blocker
br create "Session store corruption: invalid UTF-8 keys cause crash on load" \
  -t bug -p 0 \
  -d "Session store .Load() panics on keys containing invalid UTF-8 bytes. Reproducible by creating a session, manually corrupting the file..."

# Wire as a blocker (not discovered-from), so dep tree reflects reality
br dep add br-99 br-42 --type blocks
# Now br-42 is blocked by br-99
```

The current issue (`br-42`) is now blocked. Two options:

1. **Work the blocker yourself.** Update `br-42`'s notes to say "blocked on br-99; pausing here." Move to `br-99` via `br update br-99 --claim`. When `br-99` closes, `br-42` becomes ready again.
2. **Release `br-42` for someone else to pick up the blocker.** Less natural in a solo workflow; usually case 1 is the right move.

In either case, the *current* issue's status is now stale (`in_progress`, but you're not working it). Set it back to `open` or `blocked` to be honest about state:

```bash
br update br-42 --status blocked
```

(`--status blocked` is one of the rare cases where the explicit blocked status is the right answer rather than just adding a `blocks` dep edge. Use it when the blocker is filed and you want the current issue's status to reflect "waiting on something filed.")

## Checklist

When something surprises you mid-work:

1. **Pause coding.** Don't proceed past the surprise without resolving it.
2. **Categorize**: branch 1 (new bead), branch 2 (current bead wrong), branch 3 (stop and ask).
3. **For branch 1**: file the new issue (or `br q` placeholder), wire `discovered-from`, brief note in current issue, resume.
4. **For branch 2**: update the relevant field(s) on the current issue, add `br comments add` recording why, update notes, resume.
5. **For branch 3**: stop, update notes with the question and the state at stop, add audit comment, surface to user.
6. **Never silently mutate** — every change to a current issue's `acceptance` or `design` deserves a `br comments add` recording the why.
7. **Never silently absorb** — if the surprise is real work, it gets its own bead, with a `discovered-from` edge.
