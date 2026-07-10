# Bulk Import via Markdown

`br create -f <file.md>` (or `--file`) creates many issues in a single call by parsing a structured markdown file. This is the canonical path for plan-to-Beads translation when the plan describes more than ~5 related issues.

## Why bulk import for plan distillation

| Hand-scripted `br create` calls | `br create -f` |
|---|---|
| Each call creates one issue | Single call creates all issues |
| Intra-issue refs need shell variable juggling (capture ID, feed it to next call) | Intra-file refs resolve symbolically (by title or stand-in ID) |
| `--design` / `--acceptance-criteria` require follow-up `br update` calls | All fields settable inline via `### Design`, `### Acceptance` H3 sections |
| Partial failure = inconsistent state across many commands | Single command, single decision point; orchestration is simpler |
| Hard to review before running | The markdown file itself is the reviewable artifact |

**On atomicity**: `br create -f` is not transactional. It can create issues *and* emit warnings about dependency problems in the same run. Inspect the `--json` output before assuming the import landed cleanly. Do not assume all-or-nothing semantics.

**Validation tier**: For simple imports, run `br create --file --json` and inspect the output. For large or dependency-heavy imports, or when validating unfamiliar `br` behavior, test once in a temporary Beads repo before running it against the real workspace:

```bash
TMPDIR=$(mktemp -d) && cd "$TMPDIR" && br init --quiet
br create --file /path/to/import.md --json
```

Rough triggers for the temp-repo path: 20+ issues in one import; heavy use of symbolic `### ID` / `### Parent` / blocking dependencies; the `.beads/` directory is shared or about to be committed; or you're unsure whether installed `br` behavior matches this skill's docs.

**Publication scan**: when `.beads/` may be committed, pushed, shared, or made
public, scan the import markdown and resulting JSON before touching the real
workspace. Look for private paths, private module/org names, credentials,
private tag/install validation, legal/publication gates, raw prompts, agent
coordination residue, and local-only plan provenance.

Useful pattern:

```bash
rg -n -e '/Users/' -e '/home/' -e 'GOPRIVATE|GONOSUMDB|TOKEN|SECRET|PASSWORD' \
  -e 'private tag|private validation|private repo|no license|LICENSE_PENDING' \
  -e 'legal|trademark|outreach|launch gate' import.md
```

The canned pattern only catches generic residue. Before scanning, build a
short **project-specific residue list** and append it: the project's prior or
working names, private planning-doc filenames (roadmaps, parent specs), the
owner's name, and session provenance phrases ("owner ruling", "as decided",
"per the spec"). These are exactly the leaks the generic pattern misses:

```bash
rg -n -e '<old-codename>' -e 'ROADMAP|SPEC-' -e '<owner-name>' \
  -e 'owner ruling|parent spec|as decided' import.md
```

**Placement of the import file**: when the repo is public or shared, keep the
import markdown *outside the working tree* (or in an explicitly gitignored
path). It is a raw planning artifact; inside the tree, a later bulk `git add`
publishes it. Choosing its location is part of the distillation procedure,
not an afterthought.

For large imports, create in a temp repo, inspect the exported JSONL, then
discard the temp repo and only import into the real workspace after redaction.

**Before bulk-creating**: confirm `issue_prefix` matches the project (`br config get issue_prefix --json`; set with `br config set issue_prefix=<name>` if wrong — prefix changes only affect *new* issues). Never edit `.beads/issues.jsonl` by hand; all mutations go through `br` so the DB and JSONL stay in sync.

The grammar exists *so that field separation is preserved* in batch creation, not as an excuse to collapse fields together. Each issue's `### Description`, `### Design`, and `### Acceptance` sections should still separate why/how/done-when — see [field-semantics.md](field-semantics.md).

## File requirements

- Extension must be `.md` or `.markdown`.
- Parser caps file size at 10 MB.
- Symlinks and `..` path traversal are rejected.

## Grammar

````markdown
## Issue Title                  ← H2 starts a new issue
### Section Name                ← H3 starts a section within an issue
Content for that section.
````

### Recognized H3 sections (case-insensitive)

| Section | Maps to | Notes |
|---|---|---|
| `### ID` | stand-in ID for intra-file references | Not the real ID — resolved at import |
| `### Parent` | `--parent` | Accepts real ID, H2 title of another issue in the file, or stand-in ID |
| `### Priority` | `-p` | `0-4` or `P0-P4` |
| `### Type` | `-t` | task, bug, feature, epic, chore, docs, question |
| `### Description` | `-d` | Multi-line, captured in full |
| `### Design` | `--design` (applied as update) | Multi-line |
| `### Acceptance` or `### Acceptance Criteria` | `--acceptance-criteria` | Aliases parse to the same field |
| `### Assignee` | `-a` | Single name |
| `### Labels` | `-l` | Comma or whitespace separated; bullets stripped |
| `### Dependencies` or `### Deps` | `--deps` | See dependency syntax below |

Unknown H3 sections are silently ignored. Section names are case-insensitive (`### priority`, `### PRIORITY`, `### Priority` all parse identically).

Do not rely on unknown H3 sections for private notes. They are ignored by the
importer, but the import markdown itself may still be committed, archived, or
copied into `.beads/imports/` by local workflow. Keep private planning material
out of the import artifact.

### Dependency syntax in `### Dependencies`

| Form | Behavior |
|---|---|
| `br-abc123` | Bare ID, defaults to `blocks` dependency type |
| `blocks:br-abc123` | Explicit type prefix |
| `related:br-abc123`, `discovered-from:br-abc123`, `parent-child:br-abc123` | Other dependency types |
| `external:github:org/repo#123` | External reference |
| `- Title Of Another Issue In File` | Title-based intra-file reference; bullet syntax preserves the whole line as a single dep (allows titles with spaces) |
| `- db-1` (where `db-1` is a `### ID` value) | Stand-in ID intra-file reference |

Bulleted lines (`-`, `*`, `+`, with or without `[ ]` / `[x]` checkboxes) are treated as **single** dependency references — this is what enables title-based deps with spaces. Non-bulleted lines split on commas or whitespace.

## Intra-file references — the preferred path for hierarchy and deps

When issues in a single import reference each other (parent links, blocks edges), **use the intra-file reference forms below** rather than the post-create pattern of capturing IDs from `--json` output and running `br dep add` afterward. The bulk grammar exists precisely so the import file itself is the source of truth for hierarchy and dependencies — reviewable in one place, no ID-capture juggling.

If you find yourself writing:

```bash
br create -f import.md --json > created.json
A=$(jq -r '.[] | select(.title == "...") | .id' created.json)
B=$(jq -r '.[] | select(.title == "...") | .id' created.json)
br dep add "$A" "$B" --type blocks
```

...you've split the hierarchy/dep specification away from the import file. Push it back into `### Parent` and `### Dependencies` sections instead.

Two ways to reference issues defined in the same import file:

### 1. By H2 title

Use the exact title text after `## `:

````markdown
## Build Database Schema
### Type
task

## Build API Endpoints
### Dependencies
- Build Database Schema
````

### 2. By stand-in `### ID`

Assign each issue a symbolic handle, then reference it. The stand-in ID is **not** the issue's real ID — it's resolved to the generated ID during import:

````markdown
## Build Database Schema
### ID
db-1
### Type
task

## Build API Endpoints
### Dependencies
db-1
````

References to issues that already exist outside the file resolve normally via the issue ID.

## Quirks

- The first non-empty line **immediately after** the `## Title` (before any H3) is captured as the description, but **only that first line**. Always use an explicit `### Description` section for multi-line descriptions.
- Empty H3 sections are dropped silently.
- Whitespace and case within H3 headers don't matter.

## Worked example: epic + decomposition with full field separation

````markdown
## Authentication System

### Type
epic

### Priority
P1

### Labels
auth, security

### Description
Users need to authenticate before accessing secrets. Currently the API is
open to anyone with network access.

### Acceptance Criteria
- All secret endpoints require valid JWT.
- Invalid/expired tokens return 401.
- Tokens issued via /auth/login endpoint.

## Design JWT token structure

### Parent
Authentication System

### Type
task

### Priority
P1

### Description
Need to define what claims go in the JWT and token lifetime.

### Design
Use standard claims (sub, iat, exp). Add custom 'permissions' claim array.
1 hour lifetime, refresh via separate endpoint.

### Acceptance
- JWT structure documented.
- Sample token can be generated and validated.

## Implement /auth/login endpoint

### Parent
Authentication System

### Dependencies
- Design JWT token structure

### Type
task

### Priority
P1

### Description
Users need an endpoint to exchange credentials for JWT.

### Design
POST /auth/login accepts {username, password}. Validate against user store.
Return {token, expires_at} on success.

### Acceptance
- Valid credentials return 200 with JWT.
- Invalid credentials return 401.
- Endpoint documented in OpenAPI spec.
````

Run:

```bash
br create -f auth-system-issues.md --json
```

All three issues created. The two tasks are linked to the epic via `### Parent` (title-resolved). The login endpoint task is linked to the JWT design task via `### Dependencies` (title-resolved).

## Verifying the import landed

`br create -f --json` echoes every created issue **in full** — descriptions,
designs, acceptance — easily tens of KB for a mid-size import, when the
verification question is just "did it land cleanly?". Don't read the raw
output; summarize it:

```bash
br create --file import.md --json | python3 -c "
import json, sys
raw = sys.stdin.read()
i = raw.index('[{')
if raw[:i].strip(): print('PREAMBLE (check for warnings):', raw[:i].strip())
d = json.loads(raw[i:])
print('created:', len(d))
for x in d: print(x['id'], '|', x['issue_type'], '| P%s' % x['priority'], '|', x['title'][:50])
"
```

Anything printed before the JSON array is where dependency warnings surface
(the import is not transactional — issues can be created *and* warnings
emitted in the same run).

Then run three checks:

1. **Count**: created count equals the number of `## ` H2 issues you wrote.
2. **Root set**: `br ready` must return exactly the issues you intended as
   dependency roots (typically the first task of the chain). This is a
   one-shot proof the blocks-graph resolved: if a dep reference silently
   failed, the orphaned issue shows up as spuriously ready.
3. **Heaviest node**: `br show <id> --json` on the issue with the most
   dependencies; confirm every edge (parent-child plus each blocks) is
   present.

Quirk to know: `br dep tree <epic-id>` on the *epic* prints only the epic's
own line — trees render from dependents upward, not from parents downward —
which can look like a failed import. Verify from a child (`br dep tree
<child-id>`) or use the `br ready` root-set check instead.

## When to prefer bulk import vs. individual `br create`

**Use bulk import when:**
- Creating more than ~5 related issues at once.
- Distilling a planning document.
- Intra-file references would be tedious via shell variable juggling.
- You want a reviewable markdown artifact of the proposed issues before running anything.

**Use individual `br create` when:**
- Creating 1–3 issues, especially in a flowing conversation.
- Each issue needs different command-line treatment (e.g., one needs a slug, others don't).
- The script needs to react to creation output for follow-on commands.

## Verifying live behavior

Skill docs and installed `br` behavior can drift. Before relying on advanced bulk-import features, check what the local binary actually does:

```bash
br create --help
br capabilities --command create --format json
```

If `br capabilities` is unavailable on the installed version, fall back to `br create --help`, or sanity-check a small import against `--json` output and confirm fields landed where expected.

## Source of truth

The grammar is implemented in `src/util/markdown_import.rs` in the `beads_rust` repo. If live behavior diverges from this document, the parser source wins.
