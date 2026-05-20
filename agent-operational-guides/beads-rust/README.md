# Beads-Rust Best Practices

Operational guidance for AI agents using the `beads_rust` (`br`) issue tracker.

## Operational Differences vs classic Beads

- `br` is non-invasive: it never runs git commands.
- Sync is **auto-flush by default**. Successful mutating commands export to
  `.beads/issues.jsonl` automatically. `br sync --flush-only` remains useful as
  an idempotent final export check before committing, after `--no-auto-flush`,
  after disabling auto-flush in config, or during recovery.
- After `git pull` (or when JSONL is newer than the DB), run
  `br sync --import-only` to import collaborator changes.
- `.beads/beads.db` is the primary store and `.beads/issues.jsonl` is the
  export. Keep JSONL under version control; treat the DB as local unless your
  repo policy says otherwise.
- Set the issue prefix at workspace init: `br init --prefix myproj`. To set
  it on an existing workspace: `br config set id.prefix=myproj` (or
  `br config set id.prefix myproj` — both forms work). The canonical config
  key is **`id.prefix`** (not `issue_prefix`).
- `br create` accepts `title`, `-d/--description`, `-t/--type`, `-p/--priority`
  (numeric `0-4` or `P0-P4`), `-a/--assignee`, `-l/--labels`, `--parent`,
  `--deps`, and `-f/--file` (markdown bulk import). Use `br update` to add
  `--design`, `--acceptance-criteria`, and `--notes` after creation.

## Hierarchical Child IDs

When creating child issues with `--parent <ID>`, `br` generates hierarchical
IDs by appending `.N` to the parent's ID:

```bash
br create "Auth system" --type epic                              # → br-abc123
br create "Implement login" --type task --parent br-abc123       # → br-abc123.1
br create "Implement logout" --type task --parent br-abc123      # → br-abc123.2
br create "Add OAuth" --type task --parent br-abc123             # → br-abc123.3
```

The next available child number is allocated automatically; grandchildren
follow the same pattern (`br-abc123.1.2`). The hierarchy is visible in the
ID itself and queryable via `br dep tree <epic-id>` and `br show <child-id>`.

Parent assignment is re-settable at any time with `br update <child> --parent
<new-parent>` (use `--parent ''` to remove the parent link). Note: re-parenting
changes the parent-child dependency relationship but does **not** rename
existing hierarchical IDs.

### Human-readable IDs via `--slug`

Independently of hierarchy, you can embed a human-readable slug between the
prefix and the hash using `--slug`. Slugs apply to **non-hierarchical** IDs
only; child IDs ignore the slug.

```bash
br create "Fix login redirect for SSO users" --slug fix-sso-login
# → br-fix-sso-login-8cda

# Child IDs ignore --slug (the parent.N format wins):
br create "Sub-task" --parent br-abc123 --slug ignored
# → br-abc123.1 (slug discarded)
```

Slugs normalize to lowercase ASCII alphanumerics and single hyphens, capped at
48 characters. If a slug normalizes to empty (e.g., `--slug "!!!"`), the ID
falls back to the hash-only format. `--slug` and `--parent` are orthogonal:
`--slug` shapes top-level IDs; `--parent` produces hierarchical child IDs.

## Atomic Claim Workflow

Use `--claim` to atomically take ownership of an issue:

```bash
br update br-42 --claim
```

This sets `status=in_progress` AND `assignee=$BD_ACTOR` in one operation. It also validates:
- Fails if already assigned to someone else
- Fails if issue is blocked by open dependencies (use `--force` to override)

The blocked-issue check also applies to `--status in_progress` without `--claim`.

## Command Reminders

- Day-to-day: `br create`, `br show`, `br list`, `br ready`, `br update`,
  `br close`, `br dep add`.
- Bulk creation: `br create -f <markdown-file>` parses a structured markdown
  file into many issues at once. See `BULK_IMPORT.md` in this folder for the
  grammar.
- Quick capture: `br q "title"` creates an issue and prints only its ID
  (script-friendly).
- Discovery: `br capabilities --format json` returns the runtime command
  contract. Use it before any agent-driven workflow that calls unfamiliar
  commands. `br capabilities --command create --format json` returns the
  schema for one command. See `DISCOVERY.md`.
- Agent help: `br agents --add --force` writes the canonical agent workflow
  section into `AGENTS.md`. `br robot-docs guide` prints a short in-tool
  handbook.
- Claim work: `br update <id> --claim` (atomic status + assignee, validates
  not-blocked and not-already-claimed).
- Chained work: `br close <id> --suggest-next --json` returns newly unblocked
  issues so the agent can pick the next one without re-querying.
- Coordination: `br scheduler` (rank ready work with evidence),
  `br coordination status` (read-only diagnosis of stale `in_progress`
  claims).

## Output Formats

br supports four output formats:

| Flag | Use case |
|------|----------|
| `--json` | Machine-readable, stable schema. Equivalent to `--format json`. |
| `--robot` | Alias for `--json` on most commands. Some agent-canonical commands document `--robot` as the primary flag. |
| `--format toon` | Token-efficient structured output for LLM context. Decode back to JSON with `tru --decode --expand-paths safe`. |
| (default) | Rich (TTY) or Plain (piped/`NO_COLOR`). Auto-detected. |

Always use `--json` or `--robot` when parsing programmatically. Output mode
can also be defaulted via env vars: `BR_OUTPUT_FORMAT` (highest precedence)
or `TOON_DEFAULT_FORMAT` (fallback). Set `RUST_LOG=error` for routine agent
runs to keep stderr clean.

### MCP server mode

`br serve` exposes the same workspace as an MCP server over stdio, available
only in binaries built with the `mcp` feature. Use this when an MCP-native
agent benefits from tool/resource discovery; use `br --json` when a shell
pipeline is simpler. See `docs/CLI_REFERENCE.md#serve` upstream.

## Documents

- `CODE_IN_ISSUES.md`: what code belongs in issues
- `ISSUE_DESIGN.md`: how to separate description/design/acceptance/notes
- `FIELD_SEMANTICS.md`: field-by-field reference
- `EXAMPLES.md`: before/after issue writing patterns
- `BULK_IMPORT.md`: markdown grammar for `br create -f`
- `DISCOVERY.md`: using `br capabilities` to verify CLI shape at runtime
- `ISSUES_VS_PLANNING_DOCS.md`: why issues should be self-contained, not cross-reference plan files
- `WHEN_TO_CREATE_ISSUES.md`: how to file discovered work
- `SESSION_PROTOCOL.md`: session lifecycle — start, claim, work, close, flush
- `AGENT_INTEGRATION.md`: wiring best practices into agent toolchains (three-layer model + Layer 0 discovery)
- `SAFETY.md`: sync guards, history, and data protection
