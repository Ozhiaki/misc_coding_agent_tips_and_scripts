# Live Discovery: Asking `br` What It Supports

> Static documentation drifts. The CLI is the source of truth. Ask it first.

## When to use discovery

Use the discovery commands whenever an agent is about to:

- Run a `br` command it hasn't run in this codebase before.
- Generate a script that calls `br` based on memorized syntax.
- Update a rules file or skill that names specific `br` flags.

Skipping discovery is fine for routine `br ready` / `br update --claim` /
`br close` cycles where the syntax is well-known.

## The discovery commands

```bash
# Full contract
br capabilities --format json

# One command's schema
br capabilities --command create --format json
br capabilities --command "comments add" --format json
br capabilities --command "dep add" --format json
br capabilities --command "query save" --format json
br capabilities --command update --format json

# Short agent handbook (text or JSON)
br robot-docs guide
br robot-docs guide --format json

# Cached baseline content (for offline / version-skew scenarios)
br robot-docs agent-baseline --list
br robot-docs agent-baseline help/<topic>
br robot-docs agent-baseline schemas/<target>
br robot-docs agent-baseline AGENT_JOURNEY_NOTES.md

# JSON Schemas for integrations
br schema all --format json
br schema issue-details --format toon
```

### What `agent_baseline/` contains

The `agent_baseline/` payload (served via `br robot-docs agent-baseline`)
is a compiled-in snapshot of agent-facing reference material that ships
with the binary. It includes:

| Path | Content |
|---|---|
| `help/<topic>` | Per-topic short reference (commands, fields, common workflows). |
| `schemas/<target>` | One JSON Schema per `br schema` target. |
| `AGENT_JOURNEY_NOTES.md` | Narrative guidance: what to do at session start, mid-session, and close. |

**Upgrade fallback**: when an older `br` binary doesn't expose
`agent-baseline` content for a new topic that newer rules expect, the
rules should fall back to live `br capabilities --command <name>` rather
than failing. Treat the baseline as a convenience cache, not a contract.

## What `br capabilities --format json` returns

| Field | What it tells you |
|---|---|
| `contract_version` | Version stamp. Re-read this guide and your rules if it changes. |
| `recommended_entrypoints` | The canonical agent-first commands. |
| `features` | Capability flags (e.g., `mcp`, `self_update`). |
| `commands` | List of available commands and their canonical paths. |
| `global_flags` | Flags valid on every command. |
| `exit_codes` | The exit code taxonomy with categories. |
| `env_vars` | Recognized environment variables. |
| `safety` | Documented safety guarantees and boundaries. |

With `--command <PATH>`, the response adds a `command_detail` block:

| Field | What it tells you |
|---|---|
| `canonical_path` | The command's official name (after alias resolution). |
| `aliases` | Other names that resolve to this command. |
| `subcommands` | Nested commands (e.g., `dep add`, `comments add`). |
| `positionals` | Positional arguments and their types. |
| `options` | Flags, with types, defaults, and possible values. |
| `defaults` | Default values when flags are omitted. |
| `possible_values` | Enum values for restricted-vocabulary flags. |
| `examples` | Canonical usage examples. |
| `safety_notes` | Command-specific safety notes. |

## Practical patterns

### Verify before bulk-mutating

Before a large agent-driven workflow:

```bash
br capabilities --format json | jq '.contract_version'
br capabilities --command create --format json | jq '.command_detail.options[] | select(.name == "file")'
```

### Wire into agent rules

When generating `.claude/rules/beads-quality.md` or equivalent, include:

```markdown
Before running any unfamiliar br command, call:

    br capabilities --command <name> --format json

Use the returned schema to construct invocations. Do not rely on memorized
flags if the command hasn't been used in this codebase before.
```

### Detect drift in agent skills

If a `br` invocation fails with exit code 4 (validation), re-run discovery
on that command and re-derive the invocation. The likely cause is a flag
that has been renamed, deprecated, or had its possible values changed
since the agent's training data.

## `br schema` targets

`br schema <target> --format json` returns a JSON Schema for the named
shape. The full target list (use `br schema --help` if this drifts):

| Target | Shape |
|---|---|
| `all` | Bundle of every schema. |
| `issue` | The canonical issue record. |
| `issue-details` | `br show` output. |
| `issue-list` | `br list` / `br ready` output. |
| `dep-tree` | `br dep tree` output. |
| `capabilities` | The `br capabilities` envelope itself. |
| `coordination-status` | `br.coordination.v1` envelope from `br coordination status`. |
| `audit-log` | `br audit log` entries. |
| `audit-summary` | `br audit summary` aggregation. |
| `scheduler` | `br scheduler` evidence/rank output. |
| `comments` | `br comments list` output. |
| `history` | `br history list` / `diff` output. |

Use these for codegen of typed clients or for validating piped output
inside agent harnesses.

## Why not rely on `--help`?

`--help` is fine for humans. For agents:

- `br capabilities --format json` is parseable structured data, not text.
- It includes `contract_version` for change detection.
- It includes `possible_values` enums, which `--help` may not expose.
- It includes `safety_notes` per-command.

Use `--help` interactively; use `br capabilities --format json`
programmatically.
