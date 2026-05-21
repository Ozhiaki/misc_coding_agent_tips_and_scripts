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

## Commands Worth Knowing

| Family | Commands | What they're for |
|---|---|---|
| Day-to-day | `br create`, `br show`, `br list`, `br ready`, `br update`, `br close`, `br dep add` | The minimum surface. |
| Bulk creation | `br create -f <markdown-file>` | Parses a structured markdown file into many issues at once. See `BULK_IMPORT.md`. |
| Quick capture | `br q "title"` | Creates an issue and prints only its ID (script-friendly). |
| Claim work | `br update <id> --claim` | Atomic status + assignee, validates not-blocked and not-already-claimed. |
| Chained work | `br close <id> --suggest-next --json` | Returns newly unblocked issues so the agent picks the next without re-querying. |
| Discovery | `br capabilities [--command <name>] --format json`, `br robot-docs guide`, `br schema <target> --format json` | Runtime command contract, agent handbook, JSON schemas. See `DISCOVERY.md`. |
| Agent help | `br agents --add --force` | Writes the canonical agent workflow section into `AGENTS.md`. |
| Coordination | `br scheduler`, `br coordination status` | Rank ready work with evidence; read-only diagnosis of stale `in_progress` claims. |
| Queries (saved) | `br query save <name> <expr>`, `br query run <name>`, `br query list`, `br query delete <name>` | Reusable named filters; `run` accepts override flags. |
| Comments | `br comments add <id> <body>`, `br comments list <id>` | Append-only commentary on an issue — the place for traceable session-by-session history (distinct from `--notes`, which is a single mutable field). |
| Doctor | `br doctor`, `br doctor diagnose`, `br doctor repair`, `br doctor ls`, `br doctor undo <run-id>`, `br doctor explain <finding-id>` | Diagnose and (with `--repair`/`repair`) fix workspace issues. Has its own exit-code dictionary — see SAFETY and the forthcoming ERRORS_AND_SCHEMAS doc. |
| History | `br history list`, `br history diff <file>`, `br history restore <backup>`, `br history prune --keep N --older-than DAYS` | Manage `.beads/.br_history/` backups. |
| Upgrade | `br self-update` *(feature-gated)* | Available only on binaries built with the `self_update` feature; check `br capabilities --format json \| jq '.features'`. |

## Output Formats

br supports four output formats:

| Flag | Use case |
|------|----------|
| `--json` | Machine-readable, stable schema. Equivalent to `--format json`. |
| `--robot` | Alias for `--json` on most commands. Some agent-canonical commands document `--robot` as the primary flag. |
| `--format toon` | Token-efficient structured output for LLM context. Decode back to JSON with `tru --decode --expand-paths safe`. |
| (default) | Rich (TTY) or Plain (piped/`NO_COLOR`). Auto-detected. |

Always use `--json` or `--robot` when parsing programmatically. Output mode
can also be defaulted via env vars: `BR_OUTPUT_FORMAT` wins over
`TOON_DEFAULT_FORMAT` (fallback) when both are set. Set `RUST_LOG=error` for
routine agent runs to keep stderr clean.

### Other environment variables

| Variable | Effect |
|---|---|
| `BR_AGENT_NAME` | Attribution: tags mutations with the agent identity. |
| `BR_HARNESS` | Attribution: tags the harness/runner (e.g., `claude-code`). |
| `BR_MODEL` | Attribution: tags the model (e.g., `claude-opus-4-7`). |
| `BR_INHERITED_CONTEXT` | Set to `1` to enable inherited-context emission for this run (wins over config). |
| `BR_DOCTOR_RUNS_DIR` | Override the directory `br doctor` writes runs to. |

The three attribution vars feed `br audit log <id>` and are surfaced as
capture-only metadata — they never reject a command. CLI flag `--actor`
wins over env vars.

### Global flags worth knowing

A handful of flags work on every `br` command:

| Flag | Use |
|---|---|
| `--actor <name>` | Override attribution for this invocation. Precedence: `--actor` > `BR_AGENT_NAME` env > absent. Surfaced in `br audit log`. |
| `--lock-timeout <ms>` | How long to wait for a DB lock before failing. Useful in concurrent agent setups. |
| `--no-db` | Run against JSONL only, no SQLite read/write. For inspection in environments where the DB is unavailable or stale. |

### MCP server mode (optional)

`br serve` exposes the same workspace as an MCP server over stdio. **MCP is
not built into stock `br` binaries** — it's a Cargo feature (`--features mcp`)
and is opt-in. For the common CLI/JSON pipeline path you don't need it; check
`br capabilities --format json | jq '.features'` to see whether your binary
includes it. Use MCP mode when an MCP-native agent benefits from tool/resource
discovery; use `br --json` when a shell pipeline is simpler. See
`docs/CLI_REFERENCE.md#serve` upstream.

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
- `ERRORS_AND_SCHEMAS.md`: structured error envelope, the two exit-code dictionaries (ordinary vs doctor), and `br schema` targets
- `RECOVERY.md`: decision tree for sync divergence, three-way merge, force modes, JSONL conflicts, rebuild, stale-claim reclaim, and `br doctor undo`
- `AGENT_COORDINATION.md`: three coordination problems and the features that address them — `br coordination status`, inherited `agent_context`, attribution and closure-time policy gates
