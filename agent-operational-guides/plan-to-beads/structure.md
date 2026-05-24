# Structure: Epics, Hierarchy, and Dependencies

How to shape the issue tree when distilling a multi-step plan: parent-child decomposition, dependency edges between siblings, ID-shape mechanics, and when to reach for each.

## The shape choices

A planning document typically describes work at multiple scales. When distilling, you choose how to encode that structure in `br`:

| Plan-doc pattern | `br` encoding |
|---|---|
| "Big effort with multiple sub-tasks" | Epic + child tasks via `--parent` |
| "Task A must finish before Task B" | `--deps "blocks:<A>"` on Task B (or default `--deps <A>`) |
| "Found this while doing something else" | `--deps "discovered-from:<original>"` on the new issue |
| "Loosely related, not blocking" | `--deps "related:<other>"` |
| "Tracked elsewhere (GitHub, Jira, other workspace)" | `--deps "external:<service>:<id>"` |

## Hierarchical child IDs via `--parent`

When creating child issues with `--parent <ID>`, `br` generates hierarchical IDs by appending `.N` to the parent's ID:

```bash
br create "Auth system" --type epic                              # → br-abc123
br create "Implement login" --type task --parent br-abc123       # → br-abc123.1
br create "Implement logout" --type task --parent br-abc123      # → br-abc123.2
br create "Add OAuth" --type task --parent br-abc123             # → br-abc123.3
```

The next available child number is allocated automatically. Grandchildren follow the same pattern (`br-abc123.1.2`). The hierarchy is visible in the ID itself and queryable via `br dep tree <epic-id>` and `br show <child-id>`.

**Re-parenting**: `br update <child> --parent <new-parent>` moves a child to a different epic. `br update <child> --parent ''` removes the parent link. Re-parenting changes the dependency graph but does **not** rename existing hierarchical IDs — the `.N` suffix is sticky.

**Parent-child is also a dependency edge**: it shows up in `br dep tree` and `br show`. The parent isn't a separate metadata field — it's structurally the same as a typed dependency.

## Human-readable IDs via `--slug`

Independently of hierarchy, you can embed a human-readable slug between the prefix and the hash on **top-level** (non-child) issues:

```bash
br create "Fix login redirect for SSO users" --slug fix-sso-login
# → br-fix-sso-login-8cda
```

Slugs normalize to lowercase ASCII alphanumerics and single hyphens, capped at 48 characters. If a slug normalizes to empty (e.g., `--slug "!!!"`), the ID falls back to the hash-only format.

**Slugs and `--parent` are orthogonal**:
- `--slug` shapes top-level IDs.
- `--parent` produces hierarchical child IDs that ignore any provided slug.

```bash
br create "Sub-task" --parent br-abc123 --slug ignored
# → br-abc123.1 (slug discarded; child ID format wins)
```

**When to use slugs**: for top-level issues you'll reference by name often (cross-referencing in commits, mentioning in conversations). For deeply-decomposed work where hierarchical IDs already tell the story, slugs add noise. Use sparingly.

## Mixed prefixes

A repo can contain issues from multiple prefixes simultaneously (e.g., `auth-` and `pay-` in the same workspace). This is normal when consolidating issues across projects or when a subsystem owns its own prefix. `br` resolves IDs by exact match on the full prefixed form (`auth-abc123`); unprefixed lookups fail with `AMBIGUOUS_ID` if more than one prefix has a match.

## Dependency edges via `--deps`

Beyond parent-child, you wire issues together with typed dependency edges. Set at create time:

```bash
br create "Add OAuth endpoint" --type task --deps "br-abc123.1"
# Default type: blocks (so br-abc123.1 blocks this new issue)

br create "Improve error handling" --type task --deps "discovered-from:br-42"
# Type-prefixed: this issue was discovered while working br-42
```

Or post-hoc with `br dep add`:

```bash
br dep add br-99 br-42 --type discovered-from
br dep add br-99 br-100 --type related
```

### Dependency types

| Type | Meaning | When to use |
|---|---|---|
| `blocks` (default) | Other issue must complete before this one | Sequential work; A must finish before B |
| `related` | Loose association, no execution ordering | Touches similar code; useful context for reviewer |
| `discovered-from` | This issue was found while working another | Filing scope-creep discoveries cleanly |
| `parent-child` | Hierarchical (prefer `--parent`) | Rarely set directly — `--parent` does this |
| `external` | Reference to issue tracked outside this workspace | GitHub issue, Jira ticket, separate `br` workspace |

### External dependencies

External deps track work whose canonical form lives outside this `br` workspace. The `external:<service>:<id>` form is opaque to `br` — stored as a string reference, surfaced in `br dep tree` / `br show` so reviewers can follow the link. `br` does not fetch or validate the external target.

```bash
br dep add br-42 'external:github:org/repo#123' --type external
br dep add br-42 'external:jira:PROJ-456'       --type external
br dep add br-42 'external:br:other-repo/br-99' --type external
```

For cross-workspace `br` references specifically, a `.beads/routes.jsonl` file in the current workspace can map prefixes to filesystem paths so `br dep tree` knows where to point human reviewers. The routing file is a lookup table only — `br` does not mutate or sync other workspaces.

## Worked example: epic with decomposition

A plan that says "build an auth system with JWT issuance and a login endpoint" distills to:

```bash
# 1. Create the epic
br create "Authentication System" --type epic --priority 1 \
  -d "Users need to authenticate before accessing secrets. Currently the API is open to anyone with network access."
# → br-a1b2 (top-level hash ID)

br update br-a1b2 \
  --acceptance-criteria "All secret endpoints require valid JWT. Invalid/expired tokens return 401. Tokens issued via /auth/login endpoint."

# 2. Create child tasks under the epic
br create "Design JWT token structure" --type task --priority 1 --parent br-a1b2 \
  -d "Need to define what claims go in the JWT and token lifetime."
# → br-a1b2.1

br update br-a1b2.1 \
  --design "Use standard claims (sub, iat, exp). Add custom 'permissions' claim array. 1 hour lifetime, refresh via separate endpoint." \
  --acceptance-criteria "JWT structure documented. Sample token can be generated and validated."

br create "Implement /auth/login endpoint" --type task --priority 1 --parent br-a1b2 \
  --deps "br-a1b2.1" \
  -d "Users need an endpoint to exchange credentials for JWT."
# → br-a1b2.2 (blocks-depends-on br-a1b2.1)

br update br-a1b2.2 \
  --design "POST /auth/login accepts {username, password}. Validate against user store. Return {token, expires_at} on success." \
  --acceptance-criteria "Valid credentials return 200 with JWT. Invalid credentials return 401. Endpoint documented in OpenAPI spec."
```

Result: an epic with two children, the second blocked-by the first, all with proper field separation.

**For decomposition with more than ~5 children**, this create-then-update pattern becomes tedious. Switch to bulk import — see [bulk-import.md](bulk-import.md). The bulk path also lets you reference issues symbolically (by H2 title or stand-in `### ID`) so you don't have to capture generated IDs into shell variables.

## Type selection

When distilling, pick the issue type that matches the work:

| What the plan describes | `--type` |
|---|---|
| Multi-step effort with sub-tasks | `epic` (then decompose with `--parent`) |
| Concrete bounded task | `task` |
| Net-new capability | `feature` |
| Something broken | `bug` |
| Code quality / tech debt cleanup | `chore` |
| Documentation work | `docs` |
| Investigation with clear acceptance criteria | `task` (not "question" unless truly open-ended) |

Epics with no children are usually mis-typed — should they be tasks? Tasks that grow more than ~5 sub-bullets in their `design` are usually mis-typed — should they be epics?

## Priority

`br` accepts numeric (`0-4`) or letter-prefixed (`P0-P4`) priorities. They mean the same thing.

| Priority | Rough meaning |
|---|---|
| P0 | Drop everything (security, data loss, total outage) |
| P1 | High — blocks the current goal |
| P2 | Important but not urgent |
| P3 | Backlog material; would be nice |
| P4 | Speculative; "someday maybe" |

When distilling a plan, default to P2 unless the plan explicitly signals higher urgency.

## Labels

Use `-l "label1,label2"` at create time (comma-separated) or `--add-label`/`--remove-label`/`--set-labels` at update time. Labels are free-form strings. Useful for cross-cutting categorization (e.g., `auth`, `security`, `migration`, `tech-debt`) that doesn't fit the epic-task hierarchy.

Don't over-label. Three or fewer per issue is usually right.
