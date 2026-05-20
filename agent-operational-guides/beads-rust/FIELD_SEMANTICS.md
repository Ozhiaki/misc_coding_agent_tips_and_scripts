# Beads-Rust Field Semantics

Detailed reference for each text field in `br create` / `br update`.

## title (required)

**What it is**: Short identifier for the work.

**Characteristics**:
- 5-10 words ideal
- Verb + noun pattern works well: "Add X", "Fix Y", "Refactor Z"
- Should be unique enough to distinguish from similar issues

**Examples**:
- ✓ "Add Sensitive boolean to SecretObject"
- ✓ "Fix credential rotation race condition"
- ✗ "Update the code" (too vague)
- ✗ "Implement the new feature for handling sensitive data in the secret object model with proper defaults" (too long)

**Tip on ID shape**: `br` has two independent ID-shape mechanisms:

1. **Hierarchy via `--parent <ID>`**: child IDs are `<parent>.<N>`,
   e.g., `br-abc123.1`. Grandchildren extend: `br-abc123.1.2`. The next
   child number is allocated automatically.
2. **Slugs via `--slug <text>`**: top-level IDs embed the slug between
   prefix and hash, e.g., `br-fix-sso-login-8cda`. Slugs normalize to
   lowercase ASCII alphanumerics and single hyphens, capped at 48 chars.

The two are independent. A child ID ignores any `--slug` argument; the
hierarchical `<parent>.<N>` format wins.

```bash
# Hierarchy:
br create "Auth system" --type epic                          # → br-abc123
br create "Add SSO" --type task --parent br-abc123           # → br-abc123.1

# Slug on top-level:
br create "Fix login redirect for SSO users" --slug fix-sso-login
# → br-fix-sso-login-8cda

# Slug on child is discarded:
br create "Sub-task" --parent br-abc123 --slug ignored
# → br-abc123.2 (slug dropped)
```

---

## description (`-d`, `--description`)

**What it is**: The *problem statement*—why this work exists.

**Characteristics**:
- Immutable by design (problem doesn't change)
- Context for future-you after compaction
- Answers: "Why are we doing this?"

**Should contain**:
- What's wrong or missing
- Why it matters
- Relevant context/constraints

**Should NOT contain**:
- Implementation details (→ design via `br update --design`)
- Success criteria (→ acceptance via `br update --acceptance-criteria`)
- Progress updates (→ notes via `br update --notes`)

**Setting at create time**:

```bash
br create "Title" -d "Why this matters..."
```

**In bulk markdown import**: use `### Description` as an H3 section. The
first non-empty line after the H2 title (before any H3) is also captured as
description, but only that first line — subsequent lines are ignored. To
preserve multi-line descriptions, always use the explicit `### Description`
section. See `BULK_IMPORT.md`.

**Example**:
```
SecretObject needs a Sensitive field to control default sensitivity 
for new fields during creation. Currently there's no way to mark an 
entire secret as sensitive by default, forcing field-by-field marking.
```

---

## design (`--design`)

**What it is**: The *implementation approach*—how you'll solve it.

**Characteristics**:
- Mutable (update as you learn)
- Technical details belong here
- Can be empty initially, filled in during work

**Should contain**:
- Technical approach
- Key implementation decisions
- File/function targets
- Dependencies or prerequisites

**How to set**: `br update <id> --design "..."` (can be combined with other updates).

In bulk markdown import, use `### Design` as an H3 section — see `BULK_IMPORT.md`.

**Example**:
```
Add Sensitive bool to SecretObject struct in internal/model/secret.go.
Default true (fail-safe). Add json:",omitempty" tag so true values 
don't clutter JSON. Update NewSecretObject() constructor to set 
Sensitive: true.
```

---

## acceptance (`--acceptance-criteria`)

**What it is**: *Success criteria*—how you know it's done.

**Characteristics**:
- Stable (criteria shouldn't drift)
- Verifiable (can be checked)
- Answers: "How do we know this is complete?"

**Should contain**:
- Specific, testable conditions
- Observable behaviors
- Edge cases that must work

**How to set**: `br update <id> --acceptance-criteria "..."`.

In bulk markdown import (`br create -f file.md`), the section can be written
as either `### Acceptance` or `### Acceptance Criteria` — both parse to the
same field. There is no `--acceptance` short flag on `br update`.

**Format options**:
- Bullet list of criteria
- Given/When/Then scenarios
- Simple declarative statements

**Example**:
```
- SecretObject has Sensitive bool field defaulting to true
- NewSecretObject() sets Sensitive: true
- JSON output omits Sensitive field when true (omitempty)
- Existing secrets without Sensitive field default to true on load
```

---

## notes (update only)

**What it is**: *Progress log*—session checkpoints and decisions.

**Characteristics**:
- **Overwrite semantics**: `--notes` replaces existing content. Read before writing.
- Updated via `br update <id> --notes "..."`
- To preserve history, read existing notes with `br show <id> --json`, then write combined old + new.
- Survives compaction (summarized)

**Should contain**:
- Session progress
- Blockers encountered
- Decisions made and why
- Handoff context for next session

**Example**:
```bash
# Read existing notes first
br show br-42 --json | jq -r '.notes'

# Write combined old + new
br update br-42 --notes "Previous content here...
---
2024-12-27: Started implementation. Discovered Field.Sensitive already
exists—this is for SecretObject-level default only. Updated design."
```

---

## estimate (`-e`, `--estimate`)

**What it is**: Time estimate for the work.

**Characteristics**:
- Optional planning field
- Accepts minutes or human-readable format
- Useful for prioritization and capacity planning

**Format**:
```bash
br create "Fix bug" -e 30           # 30 minutes
br create "Add feature" -e 2h       # 2 hours
br create "Refactor module" -e 1d   # 1 day

# Update later if needed
br update br-42 --estimate 120
```

---

## Quick Reference

| Field | Create flag | Update flag | Mutable? | Survives Compaction? | One-liner |
|-------|-------------|-------------|----------|---------------------|-----------|
| title | positional | `--title` | Rarely | Yes | What |
| description | `-d` | `--description` | No | Yes (summarized) | Why |
| design | bulk import only | `--design` | Yes | Maybe | How |
| acceptance | bulk import only | `--acceptance-criteria` | Rarely | Yes | Done when |
| notes | not at create | `--notes` (overwrites) | Append manually | Yes (summarized) | Progress |
| estimate | `-e` | `--estimate` | Yes | Yes | How long |
| parent | `--parent` | `--parent` | Yes | Yes | Epic link |
| labels | `-l` | `--add-label`/`--remove-label`/`--set-labels` | Yes | Yes | Categorization |
| deps | `--deps` | `br dep add` | Yes | Yes | Blocking/related |
