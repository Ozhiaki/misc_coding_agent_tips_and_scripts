# Agent Coordination

> How `br` keeps multiple agents (and humans) honest when they share an
> issue tracker.

This file covers three coordination problems and the three `br` features
that address them. None of these features are on by default in a fresh
workspace — solo-dev flows see zero change unless they opt in.

## The Three Problems

| Problem | Symptom | Feature |
|---|---|---|
| **Stale claims hidden behind `in_progress`** | `br ready` shows nothing because an absent agent never closed or unclaimed. | `br coordination status` |
| **Cold-start, context decay, stale propagation** | An agent claims a child issue without ever reading the epic's governing instructions; or had them once and lost them to compaction; or read a stale snapshot. | Inherited `agent_context` |
| **Untraceable closes from anonymous agents** | An issue is closed with `--reason "Done"` and no one knows who or what closed it. | Attribution + closure-time policy gates |

Think of these as three layers of accountability: **before** work starts
(coordination status), **during** work (inherited context surfacing
governing instructions), and **at** close (attribution and policy gates
producing a traceable trail).

## 1. Stale-Claim Diagnosis (`br coordination status`)

`br coordination status` is read-only. It never mutates state and never
calls Agent Mail. It emits the `br.coordination.v1` envelope describing
each active `in_progress` claim along with advisory fields driven by
policy:

- `reclaim_allowed_by_policy` — whether the workspace's policy permits a
  third party to reclaim this issue at all
- `required_human_confirmation` — whether reclaim requires explicit human
  sign-off
- `evidence_summary` — what `br` knows about the claim (timestamps, last
  comment, Agent Mail reservation if snapshot is available)
- `suggested_commands` — recipe for the next step, often the audit-
  comment-then-reclaim sequence

```bash
br coordination status --json
```

Pass offline Agent Mail snapshots via `--reservations` / `--agents` when
network access isn't available.

`suggested_commands` is advisory, never auto-runnable. The discipline:
**audit-comment first, reclaim second**.

```bash
# Read the diagnosis
br coordination status --json | jq '.claims[] | select(.id == "br-42")'

# Audit-comment with what you know
br comments add br-42 "Reclaiming. Previous agent (claude-opus-4-6) last touched 2026-05-18; no Agent Mail heartbeat in 36h."

# Then reclaim atomically
br update br-42 --claim --force
```

See `RECOVERY.md` for the broader reclaim playbook.

## 2. Inherited `agent_context`

The problem inheritance solves is three failure modes at once:

1. **Cold-start miss** — an agent claims a child of an epic and never
   reads the epic. It works the child in isolation, missing constraints
   that live on the parent.
2. **Context decay** — an agent had the epic's instructions in context
   earlier, but compaction or session restart dropped them.
3. **Stale propagation** — the epic's instructions changed mid-flight; an
   agent still operates on a cold-start snapshot from before the update.

The fix: an `agent_context` field on each issue, plus optional inheritance
that surfaces parent + root ancestor context whenever an agent touches a
descendant.

### Defaults: off

Inheritance is **opt-in per project**. A fresh workspace stores
`agent_context` if you set it but emits nothing on descendant operations.
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

### When inheritance emits

Once enabled, inheritance surfaces blocks on exactly three commands:

- `br show <id>`
- `br update <id> --status in_progress`
- `br update <id> --claim`

The v1 emission is at most **two blocks**: the immediate parent and the
root ancestor (preferring the topmost epic in the chain; falling back to
the chain terminal if no epic exists). Tombstoned or missing ancestors
are silently skipped. Cycles stop traversal and log to stderr.

Render format (canonical text mode):

```
--- Inherited context (from epic br-a1b2: "Authentication System") ---
{ ...the agent_context content... }
--- end inherited context ---
```

JSON/TOON output wraps the same content in an `inherited_context` field
on the response envelope.

### Setting `agent_context`

```bash
# Inline JSON
br update br-a1b2 --agent-context '{
  "preferred_agents": ["claude-opus-4-7"],
  "forbidden_tools": ["bash:rm -rf"],
  "review_required": true
}'

# From a file (JSON or YAML)
br update br-a1b2 --agent-context @./agent-context.yaml

# Clear back to NULL
br update br-a1b2 --agent-context ''
```

`br` stores the content opaquely — the shape is yours. Choose a schema
that makes sense for your harness and stick to it.

### Worked example

Setting governing instructions on an epic and watching them surface on a
child:

```bash
# 1. Set the epic's agent context
br update br-a1b2 --agent-context '{
  "preferred_agents": ["claude-opus-4-7"],
  "forbidden_tools": ["bash:rm -rf"],
  "review_required": true
}'

# 2. Enable inheritance for the project (one-time)
cat >> .beads/config.yaml <<'EOF'
inherited_context:
  enabled: true
EOF

# 3. An agent claims a child of br-a1b2
br update br-a1b2.1 --claim
# stderr/stdout (text mode) is prefixed with:
# --- Inherited context (from epic br-a1b2: "Authentication System") ---
# {
#   "preferred_agents": ["claude-opus-4-7"],
#   "forbidden_tools": ["bash:rm -rf"],
#   "review_required": true
# }
# --- end inherited context ---
```

The agent reading that output can't claim "I didn't know about the
forbidden tools" — `br` surfaced them at the moment of claim.

## 3. Attribution and Closure Policy Gates

A `.beads/policy.yaml` file activates closure-time policy gates plus an
attribution capture facility. Without `policy.yaml`, every gate is inert.

### Attribution (capture-only)

Attribution is **Tier 1**: it captures who acted but **never rejects** a
command. Resolution precedence:

```
--actor <name>  >  $BR_AGENT_NAME / $BR_HARNESS / $BR_MODEL  >  absent
```

The captured values land in `br audit log <id>`:

```bash
BR_AGENT_NAME=claude-opus-4-7 BR_HARNESS=claude-code BR_MODEL=claude-opus-4-7 \
  br update br-42 --claim
br audit log br-42 --format json
```

Even with zero policy gates enabled, attribution makes "who closed this"
answerable.

### The five Phase-1 policy gates

These activate when `.beads/policy.yaml` is present and the gate is
configured. They run at close time (`br close`) and emit
`POLICY_VIOLATION` (ordinary exit code 4) on rejection.

| Gate | What it requires | Notes |
|---|---|---|
| `require_close_reason` | Minimum-length close reason; optional anchored regex. | Cheap baseline; stops `--reason "done"`. |
| `require_acceptance_criteria_satisfied` | All `- [ ]` items in the body's `## Acceptance Criteria` section must be `- [x]`. | Forces explicit acknowledgment that criteria were met. |
| `forbid_self_close_after_in_progress` | The actor who set `in_progress` cannot also close. | Forces a second pair of eyes. Combined with attribution, makes "marked own homework" trivially detectable. |
| `require_typed_references` | Close reason must contain at least one `kind:value` token from `required_kinds`. | Built-in kinds: `commit`, `pr`, `reviewer`, `investigation`, `agent-mail`, `dashboard`. Custom kinds added in config. |
| `attribution` (Tier 1) | Captures actor metadata. | **Never rejects.** Capture-only. |

### Example `policy.yaml`

```yaml
require_close_reason:
  min_length: 20
require_acceptance_criteria_satisfied: true
forbid_self_close_after_in_progress: true
require_typed_references:
  required_kinds: [commit, reviewer]
attribution:
  tier: capture

allow_bypass: true   # default; set to false to remove the escape hatch
```

### The escape hatch: `--bypass-policy`

By default `allow_bypass: true` is set. A reviewer with a documented
reason can override:

```bash
br close br-42 \
  --reason "Stale; superseded by br-99" \
  --bypass-policy \
  --bypass-reason "Closing zombie issue; no commit to cite."
```

The bypass is itself recorded in the audit log. Set `allow_bypass: false`
in `policy.yaml` to remove the escape hatch entirely.

## How They Interact in a Multi-Agent Workflow

A representative sequence with all three features active:

```bash
# Agent (with BR_AGENT_NAME=claude-opus-4-7 in env) starts a session
br sync --import-only
br ready --json

# Agent picks br-a1b2.1 (a child of the auth epic)
br update br-a1b2.1 --claim
# → inherited_context surfaces br-a1b2's agent_context
# → audit log records claim with actor=claude-opus-4-7

# Agent works, comments along the way (append-only)
br comments add br-a1b2.1 "Considered approach A, rejected (conflicts with epic's review_required: true). Going with B."

# Acceptance criteria get satisfied; agent flips the checkboxes
# in the issue body via br update --acceptance-criteria

# Close with typed references
br close br-a1b2.1 \
  --reason "Implemented JWT issuance per epic design. commit:8de732b reviewer:alice"
# → require_close_reason (>= 20 chars): pass
# → require_acceptance_criteria_satisfied: pass
# → forbid_self_close_after_in_progress: depends on who claimed
# → require_typed_references (commit + reviewer): pass

# A second agent picks up unblocked work
br close br-a1b2.1 --suggest-next --json
# → returns the next ready issue under br-a1b2

br audit log br-a1b2 --format json
# → full who-did-what trail across the epic
```

The features compose: inheritance surfaces constraints before work, the
policy gates check work was actually done, attribution makes the trail
queryable after the fact.

## Adoption Order

For a team adopting these features, the recommended sequence:

1. **Attribution first.** Set `BR_AGENT_NAME` / `BR_HARNESS` / `BR_MODEL`
   in your agent harness env. Nothing else changes — the audit log just
   becomes useful.
2. **Inheritance next, scoped to one epic.** Set `agent_context` on a
   single high-value epic. Enable `inherited_context.enabled: true`.
   Watch what surfaces. Iterate on the schema you're using.
3. **Coordination status when you hit stale claims.** First time `br
   ready` is empty for the wrong reason, run `br coordination status`.
4. **Policy gates last.** Start with `require_close_reason` and
   `require_typed_references`. Add `forbid_self_close_after_in_progress`
   once attribution is fully wired. Add
   `require_acceptance_criteria_satisfied` once issues consistently use
   the checkbox format.

Skip ahead if your project needs the harder constraints sooner. Just be
deliberate about which gates you enable and announce them to humans and
agents before flipping `allow_bypass: false`.
