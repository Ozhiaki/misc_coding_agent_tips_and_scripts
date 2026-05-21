# Beads-Rust Issue Examples

Before/after patterns showing proper field separation.

**Note:** Examples assume `id.prefix=br`.

---

## Example 1: Adding a Model Field

### ❌ Before (everything in description)

```bash
br create "Add Sensitive boolean to SecretObject model" --type task --priority 1 \
  -d "Add Sensitive bool field to SecretObject struct in internal/model/secret.go. Default value must be true (fail-safe). Add json tag with omitempty so true values do not clutter JSON output. Update NewSecretObject constructor to set Sensitive: true. This field controls the DEFAULT sensitivity for new fields during creation. Actual display uses Field.Sensitive."
```

**Problems**:
- Implementation details mixed with rationale
- No clear success criteria
- After compaction, hard to distinguish "why" from "how"

### ✓ After (proper separation)

```bash
br create "Add Sensitive boolean to SecretObject model" \
  --type task --priority 1 \
  -d "SecretObject needs a Sensitive field to control default sensitivity for new fields during creation. Currently no way to mark an entire secret as sensitive by default."

# Assume new issue ID is br-123
br update br-123 \
  --design "Add Sensitive bool to SecretObject struct in internal/model/secret.go. Default true (fail-safe). Add json:\",omitempty\" tag. Update NewSecretObject() constructor." \
  --acceptance-criteria "SecretObject has Sensitive bool defaulting to true. NewSecretObject() sets it. JSON omits field when true."
```

---

## Example 2: Bug Fix

### ❌ Before

```bash
br create "Fix race condition" --type bug --priority 0 \
  -d "There's a race condition when rotating credentials. Two goroutines can read the old credential simultaneously, both decide to rotate, and one overwrites the other's rotation. Need to add mutex or use atomic compare-and-swap. Should also add a test that spawns 100 goroutines to verify fix works."
```

### ✓ After

```bash
br create "Fix credential rotation race condition" \
  --type bug --priority 0 \
  -d "Two goroutines can read stale credential simultaneously, both trigger rotation, one overwrites the other. Results in credential loss under concurrent load."

# Assume new issue ID is br-124
br update br-124 \
  --design "Add sync.Mutex around credential read-check-rotate sequence in rotator.go. Consider RWMutex if read contention becomes issue." \
  --acceptance-criteria "No credential loss under 100 concurrent rotation attempts. Race detector passes. Existing rotation tests still pass."
```

---

## Example 3: Epic with Children

### Parent Epic

```bash
br create "Authentication System" \
  --type epic --priority 1 \
  -d "Users need to authenticate before accessing secrets. Currently the API is open to anyone with network access."

# Assume new epic ID is br-a1b2
br update br-a1b2 \
  --acceptance-criteria "All secret endpoints require valid JWT. Invalid/expired tokens return 401. Tokens issued via /auth/login endpoint."
```

### Child Tasks (with hierarchical IDs)

```bash
br create "Design JWT token structure" \
  --type task --priority 1 \
  --parent br-a1b2 \
  -d "Need to define what claims go in the JWT and token lifetime."
# → br-a1b2.1

br update br-a1b2.1 \
  --design "Use standard claims (sub, iat, exp). Add custom 'permissions' claim array. 1 hour lifetime, refresh via separate endpoint." \
  --acceptance-criteria "JWT structure documented. Sample token can be generated and validated."

br create "Implement /auth/login endpoint" \
  --type task --priority 1 \
  --parent br-a1b2 \
  -d "Users need an endpoint to exchange credentials for JWT."
# → br-a1b2.2

br update br-a1b2.2 \
  --design "POST /auth/login accepts {username, password}. Validate against user store. Return {token, expires_at} on success." \
  --acceptance-criteria "Valid credentials return 200 with JWT. Invalid credentials return 401. Endpoint documented in OpenAPI spec."
```

**Notes**:
- `--parent <ID>` at create time produces hierarchical child IDs (`<parent>.<N>`).
  This makes epic decomposition visible in the IDs themselves.
- To re-parent later, use `br update <child> --parent <new-parent>`
  (or `--parent ''` to remove the parent link). Re-parenting changes the
  dependency graph but does **not** rename existing hierarchical IDs.
- For non-parent-child dependencies (blocks, discovered-from, related),
  use `br dep add` after creation.

For multi-issue creation in a single call, see Example 6 (Bulk Import).

---

## Example 4: Refactoring

### ❌ Before

```bash
br create "Refactor storage layer" --type task --priority 2 \
  -d "The storage layer is messy. Extract interface, add better error handling, maybe add caching later."
```

**Problems**:
- Vague scope ("messy")
- Mixed concerns (interface, errors, caching)
- No clear done state

### ✓ After

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

## Pattern Summary

| Field | Ask Yourself |
|-------|--------------|
| title | "Can someone understand this in 5 words?" |
| description | "Why does this matter? What's broken/missing?" |
| design | "How will I implement this? What files/functions?" |
| acceptance | "How will I verify this is done?" |

---

## Example 5: Closing with Traceable Reasons

### ❌ Before

```bash
br close br-123 --reason "Implemented"
```

**Problem**: No trail back to code. Future agent can't verify what was done.

### ✓ After

```bash
br close br-123 --reason "Added Sensitive bool to SecretObject with true default. commit:e4f5a6b reviewer:alice"
```

**Why it works:**
- Summarizes what changed (not just "done")
- Links to the specific commit for verification
- Names the reviewer for accountability
- Stands alone after compaction

### Typed-reference vocabulary

The `kind:value` tokens (`commit:e4f5a6b`, `reviewer:alice`) are part of
`br`'s built-in typed-reference vocabulary. The built-in kinds are
`commit`, `pr`, `reviewer`, `investigation`, `agent-mail`, and `dashboard`.
They work as plain text without any configuration — close reasons are
queryable either way.

Projects that want enforcement can opt in via `.beads/policy.yaml`’s
`require_typed_references` gate, which rejects closes that lack at least
one token from a configured list of kinds. See `AGENT_COORDINATION.md`.

The upgrade path is no-cost: keep using `kind:value` now and the policy
is ready to flip on later without rewriting historic close reasons.

---

## Example 6: Bulk Import from Markdown

When creating an epic plus 5+ decomposed tasks, the create-then-update cycle
is tedious. Use `br create -f` with a structured markdown file.

### The Markdown File

Save as `auth-system-issues.md`:

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
### Type
task

### Priority
P1

### Parent
Authentication System

### Description
Need to define what claims go in the JWT and token lifetime.

### Design
Use standard claims (sub, iat, exp). Add custom 'permissions' claim array.
1 hour lifetime, refresh via separate endpoint.

### Acceptance
- JWT structure documented.
- Sample token can be generated and validated.

## Implement /auth/login endpoint
### Type
task

### Priority
P1

### Parent
Authentication System

### Dependencies
- Design JWT token structure

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

### Run the Import

```bash
br create -f auth-system-issues.md --json
```

This creates all three issues in a single call:
- The epic gets a hash ID.
- Each child task is linked to the epic via `### Parent` (referenced by H2 title).
- The `Implement /auth/login endpoint` task is linked to `Design JWT token structure` via `### Dependencies` (also referenced by title).

**Why this is better than scripting individual `br create` calls:**
- One atomic command. Partial-failure stories are simpler.
- Intra-file references resolve symbolically — no need to capture generated IDs and feed them into follow-up commands.
- Field separation (description / design / acceptance) is preserved at creation rather than backfilled.
- Easier to review the proposed issues as a markdown document before running.

See [BULK_IMPORT.md](BULK_IMPORT.md) for the full grammar.

---

## Anti-Patterns to Avoid

1. **The Wall of Text**: Everything in description
2. **The Empty Acceptance**: No way to verify completion
3. **The Premature Design**: Detailed design before understanding the problem
4. **The Vague Title**: "Fix bug" or "Update code"
5. **The Scope Creep**: Multiple unrelated concerns in one issue
