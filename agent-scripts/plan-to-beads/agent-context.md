# Setting Governing Context with `agent_context`

When a plan describes public-safe constraints that apply across all the child
tasks of an epic — preferred approaches, forbidden tools, review requirements,
schema conventions, project-specific conventions — don't repeat them on every
child issue. Set them once on the epic via `agent_context`, and let `br`'s
inheritance mechanism surface them when a descendant is later worked.

`agent_context` is exported with Beads issue data. Treat it as publishable
project state, not as a private scratchpad.

## The problem this solves

You build an epic from a plan today. Tomorrow (or next week, or after a session restart, or after context compaction) you sit down to work one of its child tasks. The epic's governing instructions — the ones that said "use approach X, not Y" or "this whole subsystem requires acceptance tests" — are gone from your context.

Three specific failure modes:

1. **Cold-start miss.** You claim a child of an epic and never read the epic. You work the child in isolation, missing constraints that live on the parent.
2. **Context decay.** You had the epic's instructions earlier, but compaction or session restart dropped them.
3. **Stale propagation.** The epic's instructions changed mid-flight; you're still operating on an older snapshot.

The fix: an `agent_context` field on each issue, plus optional inheritance that surfaces parent + root-ancestor context whenever a descendant is touched.

## Why this matters at plan-to-Beads time

`agent_context` is the only field you'd reasonably set **on the epic itself** during distillation — not on the child tasks. The whole point is that it's set once, on the governing issue, and surfaced automatically on the children later. Getting it right at creation time means future-you (after compaction, in a future session, etc.) sees the constraints without having to remember to re-read the epic first.

If you don't set it during distillation, you might never set it at all — and the public-safe constraints from the plan doc get lost.

## How to set it: `br update`, not bulk grammar

`agent_context` is set via `br update --agent-context` after the epic exists. The bulk-import markdown grammar (`br create -f`) does **not** currently parse an `### Agent Context` H3 section — such a section would be silently ignored as an unknown H3. The standard pattern when distilling:

1. Create the epic (and its children) via bulk import.
2. Capture the epic's ID from the `--json` output (or look it up by title).
3. Write the context JSON/YAML to a file (with a file-writing tool, not a shell
   heredoc), then run `br update <epic-id> --agent-context @./epic-context.json`.

If live behavior changes and bulk import gains an `### Agent Context` section, prefer that. Confirm with `br create --help` or `br capabilities --command create --format json` before relying on it.

**Current limitation**: bulk import can express issue fields, hierarchy, and dependencies, but not `agent_context`. If governing context matters, the distillation is not complete after `br create --file` — run a follow-up `br update <epic-id> --agent-context ...` on the relevant epic(s) before considering the plan-to-Beads pass done.

## Defaults: off

Inheritance is **opt-in per project**. A fresh workspace stores `agent_context` if you set it but emits nothing on descendant operations until you turn the feature on.

Two ways to enable:

```yaml
# .beads/config.yaml
inherited_context:
  enabled: true
```

```bash
# Or per-invocation via env var (wins over config)
BR_INHERITED_CONTEXT=1 br update br-a1b2.1 --claim
```

If you're setting `agent_context` on epics during plan distillation, you almost certainly want the config-file enable — otherwise the field stores fine but never surfaces.

## When inheritance emits

Once enabled, inheritance surfaces on exactly three commands:

- `br show <id>`
- `br update <id> --status in_progress`
- `br update <id> --claim`

The v1 emission is at most **two blocks**: the immediate parent and the root ancestor (preferring the topmost epic in the chain; falling back to the chain terminal if no epic exists). Tombstoned or missing ancestors are silently skipped. Cycles stop traversal and log to stderr.

### Render format (text mode)

```
--- Inherited context (from epic br-a1b2: "Authentication System") ---
{ ...the agent_context content... }
--- end inherited context ---
```

JSON/TOON output wraps the same content in an `inherited_context` field on the response envelope.

## Setting `agent_context` on an epic

```bash
# Canonical: from a file (JSON or YAML). Write the file with a file-writing
# tool, then pass it by @path:
br update br-a1b2 --agent-context @./auth-epic-context.json

# Clear back to NULL
br update br-a1b2 --agent-context ''
```

Avoid passing multi-line JSON inline in single quotes — it is shell-fragile
(zsh, for one, chokes on quote nesting in multi-line arguments and fails with
`unmatched "`). Inline is fine only for short single-line content:

```bash
br update br-a1b2 --agent-context '{"review_required": true}'
```

`br` stores the content opaquely — the shape is yours. That flexibility makes
`agent_context` easy to misuse. Keep it structured, brief, and public-safe.
Choose a schema that makes sense for the kind of constraints the plan implies
and stick to it. JSON or YAML are equally fine; YAML is often more readable when
the constraints have prose explanations.

## What goes in `agent_context`

Anything that:

- Applies across **all or most child tasks** of the epic (not specific to one child)
- Is a **constraint or convention**, not a problem statement (problem statement goes in `description`)
- Would be **costly to rediscover** if it had to be re-derived from scratch in a future session

Good candidates:

- Preferred technical approaches the plan settled on (and why alternatives were rejected)
- Forbidden patterns or tools (`rm -rf`, certain libraries, deprecated APIs)
- Review or approval requirements
- Schema conventions, naming conventions, layout conventions
- Performance baselines or quality bars
- Cross-cutting concerns (logging format, error-handling style, observability)

Poor candidates:

- Problem statement — goes in `description` instead
- Per-task implementation details — go in each child's `design` instead
- Single-use facts that apply to only one child — put them on that child directly
- Anything you're not sure will still be relevant in a week — let it surface organically through `notes` or `comments` instead

Never put these in `agent_context`:

- local absolute paths, usernames, machine names, temp/build directories, or
  private workspace layout;
- credentials, secret names/values, token handling, auth validation procedures,
  private registry/module settings, or credential-helper details;
- legal, licensing, trademark, outreach, customer, repo-visibility, or launch
  deliberations unless the issue is explicitly about public policy text;
- private tag validation, private CI/release rehearsals, or commands that only
  work with private access;
- raw prompts, session summaries, subagent names, tool routing, mailbox/thread
  IDs, or compaction notes;
- historical org/module/package names that are not intentionally public.

If the only reason to keep a fact is "future private-session continuity", keep
it outside Beads or rewrite it into a public-safe constraint.

## Worked example

A plan says: "Build out the auth system. Use JWT with RS256 (we rejected HS256 due to key-rotation pain). Never put JWTs in localStorage. Every PR needs a security review before merge. Coverage must stay above 90% in the auth module."

Distilling:

```bash
# 1. Create the epic with description (the why)
br create "Authentication System" --type epic --priority 1 \
  -d "Users need to authenticate before accessing secrets. Currently the API is open to anyone with network access."
# → br-a1b2

br update br-a1b2 \
  --acceptance-criteria "All secret endpoints require valid JWT. Invalid/expired tokens return 401. Tokens issued via /auth/login endpoint."

# 2. Set the governing context (the constraints, separate from the why).
#    Write the JSON to a file first (file-writing tool, not a heredoc):
#    auth-epic-context.json:
#      {
#        "token_format": "JWT with RS256",
#        "rejected_alternatives": ["HS256 (key-rotation pain)"],
#        "forbidden_storage": ["localStorage", "sessionStorage"],
#        "review_required": "security review on every PR before merge",
#        "coverage_floor": "90% in auth/"
#      }
br update br-a1b2 --agent-context @./auth-epic-context.json

# 3. Enable inheritance for the project (one-time; guarded so a re-run
#    doesn't append a duplicate key)
grep -q '^inherited_context:' .beads/config.yaml 2>/dev/null || \
  printf 'inherited_context:\n  enabled: true\n' >> .beads/config.yaml

# 4. Decompose the epic into children
br create "Design JWT token structure" --type task --parent br-a1b2 \
  -d "Need to define what claims go in the JWT and token lifetime."
# → br-a1b2.1

# ...further child tasks...
```

Now, in a future session — possibly weeks later, possibly after compaction — when you sit down to work `br-a1b2.1`:

```bash
br update br-a1b2.1 --claim
# stdout/stderr (text mode) is prefixed with:
# --- Inherited context (from epic br-a1b2: "Authentication System") ---
# {
#   "token_format": "JWT with RS256",
#   "rejected_alternatives": ["HS256 (key-rotation pain)"],
#   "forbidden_storage": ["localStorage", "sessionStorage"],
#   "review_required": "security review on every PR before merge",
#   "coverage_floor": "90% in auth/"
# }
# --- end inherited context ---
```

You can't claim "I didn't know HS256 was off the table" — `br` surfaced it at the moment of claim.

## What `agent_context` is *not*

- **Not a substitute for `description`.** The "why this work exists" still goes in description. `agent_context` is "how this work must be done."
- **Not a substitute for `design`.** Per-child implementation approaches still go in each child's `design` field. `agent_context` is for cross-cutting constraints.
- **Not a substitute for reading the epic.** Inheritance surfaces a snapshot at the moment of claim/show. It doesn't replace reading the epic if the epic's `description` or `acceptance` matters to the child.

## Schema choice

`br` doesn't impose a schema — the content is opaque. But picking a consistent shape across epics makes it easier to scan. A reasonable starting schema:

```yaml
constraints:
  - <hard rules>
preferences:
  - <soft guidance>
rejected:
  - <approach>: <why rejected>
context:
  - <public, durable background>
```

Stick with one schema across a project. Mixing schemas makes the inherited blocks hard to read at a glance.
