# Field Semantics

Detailed reference for each field involved in plan-to-Beads translation. The four-field content model (`description` / `design` / `acceptance` / `notes`) is the heart of writing good issues — get it right and downstream agents can work from `br show` alone.

## The model at a glance

| Field | Create flag | Update flag | Purpose | Mutable? |
|---|---|---|---|---|
| **title** | positional | `--title` | What (short name) | Rarely |
| **description** | `-d` / `--description` | `--description` | Why (problem statement) | No |
| **design** | bulk import only (`### Design`) | `--design` | How (implementation approach) | Yes, often |
| **acceptance** | bulk import only (`### Acceptance`) | `--acceptance-criteria` | Done when (success criteria) | Rarely |
| **notes** | not at create | `--notes` (overwrites) | Progress log | Each session |
| **estimate** | `-e` / `--estimate` | `--estimate` | Time estimate | Yes |
| **parent** | `--parent <ID>` | `--parent <ID>` | Epic membership | When restructured |
| **labels** | `-l` | `--add-label`/`--remove-label`/`--set-labels` | Categorization | Yes |
| **deps** | `--deps "type:id,..."` | `br dep add` | Blocking/related/discovered-from | When discovered |

Note: `br create` (single-issue mode) does **not** accept `--design`, `--acceptance-criteria`, or `--notes`. Either use `br update` after creation, or use bulk markdown import (`br create -f file.md`) which accepts `### Design` and `### Acceptance` sections inline. See [bulk-import.md](bulk-import.md).

## Why field separation matters

### 1. Compaction survival

When context compacts, br issues get summarized. If everything is in description, the summarizer can't distinguish problem from solution from criteria. Well-structured issues survive compaction with meaning intact:

- Description → "Why did we create this?"
- Acceptance → "How do we know it's done?"

### 2. Design evolution

Implementation approaches change as you learn. The `design` field is explicitly mutable — use `br update` to evolve it without polluting the immutable problem statement.

### 3. Verification

Acceptance criteria in their own field can be checked mechanically. "Does the implementation satisfy each criterion?" becomes tractable.

### 4. Bulk-creation discipline

When distilling a planning document into many issues, `br create -f` accepts all the fields inline via `### Description`, `### Design`, `### Acceptance` sections. The grammar exists *so that field separation is preserved* in batch creation — not as an excuse to collapse fields together.

---

## title

**What it is**: Short identifier for the work.

**Characteristics**:
- 5-10 words ideal
- Verb + noun pattern works well: "Add X", "Fix Y", "Refactor Z"
- Specific enough to distinguish from similar issues

**Examples**:
- ✓ "Add Sensitive boolean to SecretObject"
- ✓ "Fix credential rotation race condition"
- ✗ "Update the code" (too vague)
- ✗ "Fix auth" → ✓ "Fix OAuth implicit grant → PKCE in login flow"
- ✗ "Implement the new feature for handling sensitive data in the secret object model with proper defaults" (too long)

For details on ID shape (`--parent` hierarchical IDs vs. `--slug` top-level IDs), see [structure.md](structure.md).

---

## description (`-d` / `--description`)

**What it is**: The *problem statement* — why this work exists.

**Characteristics**:
- Immutable by design (the problem doesn't change)
- Provides context for future-you after compaction
- Answers: "Why are we doing this?"

**Should contain**:
- What's wrong or missing
- Why it matters
- Relevant context/constraints

**Should NOT contain**:
- Implementation details (those go in `design`)
- Success criteria (those go in `acceptance`)
- Progress updates (those go in `notes`)
- References to outside planning documents (the issue must stand alone)

**Setting at create time**:

```bash
br create "Title" -d "Why this matters..."
```

**In bulk markdown import**: use `### Description` as an H3 section. The first non-empty line immediately after the H2 title (before any H3) is also captured as description, but **only that first line** — subsequent lines are ignored. Always use the explicit `### Description` section for multi-line descriptions.

**Example**:

```
SecretObject needs a Sensitive field to control default sensitivity
for new fields during creation. Currently there's no way to mark an
entire secret as sensitive by default, forcing field-by-field marking.
```

### Banned phrases

These phrases signal a non-self-contained issue. Never use them in `description`:

- "As discussed" / "as mentioned" / "per our conversation"
- "Per the spec" / "see the plan" / "reference: planning-doc.md"
- "The auth issue from earlier" / "the thing we talked about"
- "You know what I mean" / "the usual approach"

**The test**: Could a fresh agent, with zero session history, understand and implement this issue from `br show` alone? If any phrase requires shared context to parse, rewrite it with the actual information inline.

---

## design (`--design`)

**What it is**: The *implementation approach* — how you'll solve it.

**Characteristics**:
- Mutable (update as you learn)
- Technical details belong here
- Can be empty initially, filled in during work
- Code snippets are welcome here (see below)

**Should contain**:
- Technical approach
- Key implementation decisions
- File/function targets
- Dependencies or prerequisites

**How to set**: `br update <id> --design "..."` (can be combined with other updates). In bulk markdown import, use `### Design` as an H3 section.

**Example**:

```
Add Sensitive bool to SecretObject struct in internal/model/secret.go.
Default true (fail-safe). Add json:",omitempty" tag so true values
don't clutter JSON output. Update NewSecretObject() constructor to set
Sensitive: true.
```

### Code in design

`design` is one of two fields where code snippets are appropriate (the other is `notes`). Include code when it would save a future agent significant rediscovery time:

- API integration with non-obvious query patterns
- Working code that took trial-and-error to get right
- "Occult" knowledge (undocumented behavior, quirks)

Skip code for standard patterns, obvious-from-description code, or single-session work.

Code **never** belongs in `description` (problem statement, immutable) or `acceptance` (success criteria, not implementation).

---

## acceptance (`--acceptance-criteria`)

**What it is**: *Success criteria* — how you know it's done.

**Characteristics**:
- Stable (criteria shouldn't drift)
- Verifiable (can be checked)
- Answers: "How do we know this is complete?"

**Should contain**:
- Specific, testable conditions
- Observable behaviors
- Edge cases that must work

**How to set**: `br update <id> --acceptance-criteria "..."`.

In bulk markdown import, the section can be written as either `### Acceptance` or `### Acceptance Criteria` — both parse to the same field. There is no `--acceptance` short flag on `br update`.

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

If you can't write clear acceptance criteria, you don't understand the problem yet. Investigate more, or note open questions in `design`/`notes` for follow-up.

---

## notes (update only)

**What it is**: *Progress log* — session checkpoints and decisions.

**Characteristics**:
- **Overwrite semantics**: `--notes` replaces existing content. Read before writing.
- Updated via `br update <id> --notes "..."`
- To preserve history, read existing notes with `br show <id> --json`, then write combined old + new
- Survives compaction (summarized)

**Should contain**:
- Session progress
- Blockers encountered
- Decisions made and why
- Handoff context for the next session

**Read-before-write pattern**:

```bash
# Read existing notes first
br show br-42 --json | jq -r '.notes'

# Write combined old + new
br update br-42 --notes "Previous content here...
---
2026-05-23: Started implementation. Discovered Field.Sensitive already
exists — this is for SecretObject-level default only. Updated design."
```

---

## notes vs. `br comments` — when to reach for which

`br comments add` is the append-only sibling to `--notes`. Same conceptual territory (session-by-session progress) but different mechanics.

| Use `--notes` | Use `br comments add` |
|---|---|
| Current state, handoff context | Session log, decisions made, what was tried |
| Single source of truth for "where am I" | Append-only audit trail |
| Frequently overwritten | Never overwritten |

**Commands**:

```bash
br comments add br-42 "Tried approach X, hit limit Y. Switching to Z."
br comments list br-42                # newest-first by default
br comments list br-42 --format json  # machine-readable
```

For the plan-to-Beads phase specifically: usually neither matters yet at creation time. They become relevant once an agent starts working the issue. Worth knowing about so you don't try to stuff session history into `description`.

---

## estimate (`-e` / `--estimate`)

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

## Worked examples

### Example 1: Adding a model field

**❌ Before** (everything in description):

```bash
br create "Add Sensitive boolean to SecretObject model" --type task --priority 1 \
  -d "Add Sensitive bool field to SecretObject struct in internal/model/secret.go. Default value must be true (fail-safe). Add json tag with omitempty so true values do not clutter JSON output. Update NewSecretObject constructor to set Sensitive: true. This field controls the DEFAULT sensitivity for new fields during creation. Actual display uses Field.Sensitive."
```

Problems: implementation details mixed with rationale, no clear success criteria, after compaction hard to distinguish "why" from "how".

**✓ After** (proper separation):

```bash
br create "Add Sensitive boolean to SecretObject model" \
  --type task --priority 1 \
  -d "SecretObject needs a Sensitive field to control default sensitivity for new fields during creation. Currently no way to mark an entire secret as sensitive by default."

# Assume new issue ID is br-123
br update br-123 \
  --design "Add Sensitive bool to SecretObject struct in internal/model/secret.go. Default true (fail-safe). Add json:\",omitempty\" tag. Update NewSecretObject() constructor." \
  --acceptance-criteria "SecretObject has Sensitive bool defaulting to true. NewSecretObject() sets it. JSON omits field when true."
```

### Example 2: Bug fix

**❌ Before**:

```bash
br create "Fix race condition" --type bug --priority 0 \
  -d "There's a race condition when rotating credentials. Two goroutines can read the old credential simultaneously, both decide to rotate, and one overwrites the other's rotation. Need to add mutex or use atomic compare-and-swap. Should also add a test that spawns 100 goroutines to verify fix works."
```

**✓ After**:

```bash
br create "Fix credential rotation race condition" \
  --type bug --priority 0 \
  -d "Two goroutines can read stale credential simultaneously, both trigger rotation, one overwrites the other. Results in credential loss under concurrent load."

# Assume new issue ID is br-124
br update br-124 \
  --design "Add sync.Mutex around credential read-check-rotate sequence in rotator.go. Consider RWMutex if read contention becomes issue." \
  --acceptance-criteria "No credential loss under 100 concurrent rotation attempts. Race detector passes. Existing rotation tests still pass."
```

### Example 3: Refactoring (vague → specific)

**❌ Before**:

```bash
br create "Refactor storage layer" --type task --priority 2 \
  -d "The storage layer is messy. Extract interface, add better error handling, maybe add caching later."
```

Problems: vague scope ("messy"), mixed concerns (interface + errors + caching), no clear done state.

**✓ After**:

```bash
br create "Extract Storage interface from concrete implementation" \
  --type task --priority 2 \
  -d "Storage logic is tightly coupled to SQLite. Can't unit test handlers without real database. Need interface for mocking."

# Assume new issue ID is br-200
br update br-200 \
  --design "Define Storage interface in internal/storage/storage.go. Move SQLite implementation to internal/storage/sqlite/. Update all callers to use interface." \
  --acceptance-criteria "Storage interface exists. SQLite implements it. Handler tests use mock storage. No direct sqlite imports outside storage package."
```

---

## Anti-patterns to avoid

| Anti-pattern | Why bad | Fix |
|---|---|---|
| **The Wall of Text** | Everything dumped in `description` | Split into description/design/acceptance |
| **The Empty Acceptance** | No way to verify completion | Write acceptance criteria; if you can't, investigate more |
| **The Premature Design** | Detailed `design` before understanding the problem | Write `description` and `acceptance` first; let `design` grow |
| **The Vague Title** | "Fix bug", "Update code" | Be specific: verb + noun + context |
| **The Scope Creep** | Multiple unrelated concerns in one issue | Split into separate issues; link with `--deps` if related |
| **The Cross-Reference** | "See plan-v2.md for details" | Distill the plan into the issue's fields; archive the plan |
| **Code in description** | Confuses problem statement with implementation | Code goes in `design` or `notes`, never `description` or `acceptance` |
