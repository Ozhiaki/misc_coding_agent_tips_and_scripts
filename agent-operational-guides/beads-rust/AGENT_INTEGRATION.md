# Agent Integration

> How to deploy these best practices so agents actually follow them.

## The Problem

Documentation that agents read once at session start gets treated as suggestions. Quality standards need to be loaded as **constraints**, not reference material.

Think of it like the difference between a style guide on a shelf vs. a linter in CI. Both say the same thing; only one enforces it.

## The Layered Model

```
Layer 0: Live Discovery      → ASK br what it supports (capabilities, schema)
Layer 1: Workflow context    → HOW to use br (commands, session protocol)
Layer 2: Quality constraints → WHAT standards apply (field requirements)
Layer 3: Validation          → WHEN to check (post-creation, pre-push)
```

### Layer 0: Live Discovery

Before any agent calls a `br` command in a new project, the agent should ask
`br` itself what it supports. This is more reliable than reading static docs,
because the CLI is the source of truth and may be ahead of any documentation.

The two discovery entry points:

```bash
br capabilities --format json                       # Full contract
br capabilities --command create --format json      # One command's schema
br robot-docs guide --format json                   # Short agent handbook
```

`br capabilities` returns `contract_version`, `recommended_entrypoints`,
`features`, `commands`, `global_flags`, `exit_codes`, `env_vars`, and
`safety` notes. With `--command`, it adds `command_detail` with canonical
path, aliases, subcommands, positionals, options, defaults, possible values,
examples, and safety notes for that command.

Treat the version field in your generated rules. If `contract_version` jumps,
re-read this layer before relying on memorized flags.

See [DISCOVERY.md](DISCOVERY.md) for details.

### Layer 1: Workflow Context

The br SKILL.md or AGENTS.md (`br agents --add --force`) tells agents what commands exist and how to use them. Keep it focused on mechanics. Don't mix in quality rules here — agents treat session-start context as optional reference.

**Where it lives**: `.beads/AGENTS.md`, SKILL.md, or equivalent.

### Layer 2: Quality Constraints

Quality rules belong where your agent loads **hard constraints**:

| Agent | Location |
|-------|----------|
| Claude Code | `.claude/rules/beads-quality.md` |
| Cursor | `.cursorrules` |
| Codex, Copilot, Gemini CLI, Windsurf, Aider | `AGENTS.md` (read natively) |

**Why separate from Layer 1?** When workflow context and quality standards share a file, agents treat creation standards as optional reference rather than hard constraints. Separation makes constraints persistent.

### Layer 3: Validation

`br lint` checks for missing template sections by type. Wire it into your workflow:

```bash
# Manual check (defaults to open issues)
br lint

# Filter by type
br lint --type feature

# As a git pre-push hook
# In .git/hooks/pre-push or lefthook.yml:
br lint
```

Catches what slipped through Layers 1 and 2.

## Example Quality Rules

These are the constraints to put in Layer 2 (your agent's rules file):

```markdown
## Beads Quality Rules

### Creation Standards
- Descriptions must be self-contained. Banned phrases: "as discussed",
  "per the spec", "see above", "the issue from earlier", or any
  session-dependent reference. A fresh agent with zero context must be
  able to work the issue from `br show` alone.
- Always add `--design` for features and tasks (via `br update`).
- Always add `--acceptance-criteria` for bugs and features (via `br update`).
- Titles must be specific. "Fix auth" → "Fix OAuth implicit grant → PKCE
  in login flow."

### Close Discipline
- Close with `--reason` including commit ref and summary of what changed.
- Never close with vague reasons like "Done" or "Implemented."

### Notes Discipline
- `--notes` overwrites. Always read existing notes before updating.
- Include previous notes content when writing new notes.

### Dependency Wiring
- Link related issues with `br dep add`.
- When discovering bugs during feature work, create a bug issue and link
  it with `br dep add <bug> <original> --type discovered-from`.

### CLI Discovery
- Before running any unfamiliar br command, call:
    br capabilities --command <name> --format json
  Use the returned schema to construct invocations. Do not rely on memorized
  flags if the command hasn't been used in this codebase before.
```

## Verifying the Setup

After wiring in all four layers, test with a fresh agent session:

1. Does the agent call `br capabilities` before unfamiliar commands? (Layer 0 working)
2. Does the agent run `br ready` or `br list` at session start? (Layer 1 working)
3. Does the agent create issues with separate description/design/acceptance? (Layer 2 working)
4. Does `br lint --status=open` pass after the agent creates issues? (Layer 3 working)

If Layer 2 fails, the usual cause is the rules file not being in the right location for your agent tool.

## Cross-Workspace Routing

When an agent works across multiple `br` workspaces (different repos, sub-
projects, or per-team workspaces), `.beads/routes.jsonl` configures which
workspace handles which prefix. This is a routing manifest, not automatic
multi-repo sync — each workspace remains independent. `br` reads the
manifest to resolve cross-workspace IDs in dependency edges and command
output; it does **not** open or mutate other workspaces' databases.

```jsonl
{"prefix": "auth", "path": "/Users/dave/p/auth-service"}
{"prefix": "pay",  "path": "/Users/dave/p/payment-service"}
```

When an issue in this workspace declares an `external:br:auth/auth-42`
dependency, the routing manifest tells `br dep tree` and `br show` where
to point the human. Resolving the dependency for the agent still requires
`cd` to the other workspace; the manifest is purely a lookup table.

## Attribution via `--actor`

`br` accepts an `--actor <name>` global flag that overrides agent identity
for a single invocation. Precedence: `--actor` > `BR_AGENT_NAME` env >
absent. The actor is captured in `br audit log <id>` so multi-agent and
human-plus-agent workflows have a queryable trail of who did what.

```bash
br update br-42 --actor alice --claim          # attributed to alice
br audit log br-42                              # see the trail
br audit log br-42 --format json                # machine-readable
```

This is a **capture-only** facility — attribution never causes a command
to fail. Closure-time policy gates (`.beads/policy.yaml`) can additionally
*require* certain attribution fields; see the forthcoming
`AGENT_COORDINATION.md`.
