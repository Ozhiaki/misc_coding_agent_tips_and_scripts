# Beads-Rust Issue Design Philosophy

> This document teaches agents *how* to create good beads_rust (br) issues, not just *what* commands exist.

## The Problem

Agents learn br commands from AGENTS.md (install with `br agents --add --force`) or `br create --help`. But knowing the commands doesn't mean using them well.

**Common failure mode**: Stuffing everything into `--description`:

```bash
br create "Add Sensitive field" --type task --priority 1 \
  -d "Add Sensitive bool to SecretObject. Default true. Add json tag. Update constructor. Field controls default sensitivity."
```

This conflates *why* (the problem), *how* (the solution), and *done when* (success criteria) into one blob.

## The Fields

Beads-rust provides distinct fields for distinct purposes:

| Field | Create flag | Update flag | Purpose | Changes? |
|-------|-------------|-------------|---------|----------|
| **title** | positional | `--title` | What (short name) | Rarely |
| **description** | `-d` | `--description` | Why (problem/context) | Never |
| **design** | bulk import only (`### Design`) | `--design` | How (implementation approach) | Often |
| **acceptance** | bulk import only (`### Acceptance`) | `--acceptance-criteria` | Done when (success criteria) | Rarely |
| **notes** | not at create | `--notes` | Progress log | Each session |
| **parent** | `--parent <ID>` | `--parent <ID>` | Epic membership | When restructured |
| **labels** | `-l, --labels "a,b,c"` | `--add-label`, `--remove-label`, `--set-labels` | Categorization | When relevant |
| **deps** | `--deps "type:id,..."` | `br dep add` | Blocking/discovered-from/related | When discovered |

**Notes:**
- `br create` (single-issue mode) does not accept `--design`, `--acceptance-criteria`, or `--notes`. Use `br update` after creation, or use the bulk markdown import (`br create -f file.md`) which accepts `### Design` and `### Acceptance` sections inline.
- `br update --acceptance-criteria` is the long form; there is no short `--acceptance` flag — the markdown bulk import accepts `### Acceptance` as an alias for `### Acceptance Criteria`.

## Why This Matters

### 1. Compaction Survival

When context compacts, br issues get summarized. If everything is in description, the summarizer can't distinguish problem from solution from criteria.

Well-structured issues survive compaction with meaning intact:
- Description → "Why did we create this?"
- Acceptance → "How do we know it's done?"

### 2. Design Evolution

Implementation approaches change as you learn. The `--design` field is explicitly mutable—use `br update` to evolve it without polluting the immutable problem statement.

### 3. Verification

Acceptance criteria in their own field can be checked mechanically. "Does the implementation satisfy each criterion?" becomes tractable.

### 4. Bulk Creation Path

When creating many issues at once (epics + decomposed tasks, imported
backlogs, plan-distillation), `br create -f <markdown-file>` accepts all
the fields above inline. This avoids the create-then-update-then-update
cycle for each issue. See `BULK_IMPORT.md` in this folder for the grammar.

Importantly, even when using bulk import, the field discipline still
applies: each issue's `### Description`, `### Design`, and `### Acceptance`
sections should separate why/how/done-when. The grammar exists *so* that
field separation is preserved in batch creation, not as an excuse to
collapse fields together.

## The Analogy

Think of it like a user story card:

- **Front of card** (visible, stable): Title + Description + Acceptance
- **Back of card** (working notes, evolving): Design + Notes

The front answers "what are we doing and why?" The back answers "how are we doing it?"

## Banned Phrases in Descriptions

These phrases signal a non-self-contained issue. Never use them:

- "As discussed" / "as mentioned" / "per our conversation"
- "Per the spec" / "see the plan" / "reference: planning-doc.md"
- "The auth issue from earlier" / "the thing we talked about"
- "You know what I mean" / "the usual approach"

**The test**: Could a fresh agent, with zero session history, understand and implement this issue from `br show` alone? If any phrase requires shared context to parse, rewrite it with the actual information.

## When Creating Issues

Ask yourself:

1. **Title**: Can someone understand the work from 5 words?
2. **Description**: If I forgot everything else, would this explain *why* this matters?
3. **Design**: Is this *how* I'll solve it, separate from *what* I'm solving?
4. **Acceptance**: Could someone else verify completion from these criteria alone?

## See Also

- [Field Semantics](FIELD_SEMANTICS.md) - Detailed breakdown of each field
- [Examples](EXAMPLES.md) - Before/after patterns
- [Bulk Import](BULK_IMPORT.md) - Markdown grammar for `br create -f`
