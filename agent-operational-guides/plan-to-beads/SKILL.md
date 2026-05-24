---
name: plan-to-beads
description: Translate a planning document (markdown spec, design doc, requirements list, brainstorm) into well-structured beads_rust (`br`) issues — epics, tasks, dependencies, and properly-separated fields. Use when an agent has a plan artifact in hand and needs to construct issues from it, when decomposing an epic into child tasks, when populating description/design/acceptance/notes correctly, or when bulk-importing many related issues at once. NOT for working an already-created issue (claim → implement → close); that is a separate concern.
---

# Plan to Beads

This skill covers the **distillation step**: taking a planning artifact and turning it into a structured set of `br` issues that a future agent can work without re-reading the plan.

## When this skill applies

- An agent has a plan, spec, design doc, or set of requirements in hand and needs to create issues from it.
- An agent is decomposing an epic into child tasks.
- An agent is filling out an issue's fields and needs to decide what goes where.
- An agent is bulk-importing many related issues from a single markdown file.

This skill does **not** cover working an already-existing issue (claiming, implementing, closing). That belongs to a separate execution-phase skill.

## The core principle

**Issues are the work. Planning documents are the thinking that produced the work.**

After distillation, the issue must stand alone. A future agent picking it up from `br ready` — with zero session history, possibly after compaction — must be able to implement it from `br show` alone. If they would need to re-read the plan doc, the issue is under-specified and the fix is to improve the issue, not to add a cross-reference.

Concretely this rules out phrases like:
- "As discussed" / "per our conversation"
- "Per the spec" / "see plan-v2.md"
- "The auth issue from earlier"
- "The usual approach"

The test: could a fresh agent implement this from `br show` alone? If no, rewrite with the actual information inline.

## The four-field content model

This is the heart of the skill. `br` provides distinct fields for distinct purposes. Conflating them — especially stuffing everything into `description` — is the single most common failure mode.

| Field | Answers | Mutable? | Survives compaction? |
|---|---|---|---|
| **title** | What (short name, ~5 words) | Rarely | Yes |
| **description** | *Why* (problem statement, context, motivation) | No | Yes (summarized) |
| **design** | *How* (implementation approach) | Yes, often | Maybe |
| **acceptance** | *Done when* (verifiable success criteria) | Rarely | Yes |
| **notes** | Progress log, session checkpoints | Each session (overwrite) | Yes (summarized) |

**The user-story-card analogy:**

- **Front of card** (visible, stable): title + description + acceptance — "what are we doing and why?"
- **Back of card** (working notes, evolving): design + notes — "how are we doing it?"

For the full breakdown including examples, banned anti-patterns, and the distinction between `--notes` (mutable handoff snapshot) and `br comments add` (append-only audit trail), see [field-semantics.md](field-semantics.md).

## When to use bulk import vs. individual `br create`

The decision hinges on **how many related issues you're creating** and **whether they reference each other**.

**Use `br create -f <markdown-file>`** when:
- Creating more than ~5 related issues at once.
- Distilling a planning document into an epic + decomposed tasks.
- Issues reference each other (parent links, dependencies) — intra-file references resolve symbolically, no shell-variable juggling.
- You want a reviewable markdown artifact before any issues are actually created.

**Use individual `br create` calls** when:
- Creating 1–3 issues, especially in a flowing conversation.
- Each issue needs different command-line treatment.
- The script needs to react to creation output (e.g., capturing the new ID into a shell variable for follow-on commands).

Bulk import preserves field separation at creation time — `### Description`, `### Design`, `### Acceptance` parse to the right fields directly, avoiding the create-then-update-then-update cycle. See [bulk-import.md](bulk-import.md) for the full grammar.

## Epic and dependency structure

When the plan describes multi-step work, the structure of the resulting issue tree matters:

- **`--parent <ID>`** at create time produces hierarchical child IDs (`<parent>.<N>`, e.g., `br-abc123.1`). Use this when decomposing an epic into tasks. The hierarchy is visible in the ID itself.
- **`--deps`** at create time, or `br dep add` post-hoc, wires non-parent-child relationships: `blocks` (default), `discovered-from`, `related`, `external`.
- **`--slug`** shapes top-level IDs only (`br-fix-sso-login-8cda`). Children ignore slugs. Use slugs sparingly — they're for issues you'll reference by name often.

See [structure.md](structure.md) for ID-shape mechanics, dependency types, and patterns for decomposing epics.

## Setting governing context on epics

When a plan implies constraints that apply across all its child tasks (preferred approaches, forbidden tools, review requirements, schema conventions), don't repeat them on every child issue — set them once on the epic via `agent_context`, and let `br`'s inheritance surface them when an agent later claims a child.

This is especially valuable for solo work across sessions: future-you (after compaction or session restart) will see the epic's governing instructions when claiming any descendant, without having to remember to read the epic first.

See [agent-context.md](agent-context.md) for the schema, the per-project enable flag, and how inheritance surfaces.

## Quick checklist before creating any issue

1. **Title**: Can someone understand the work from 5 words? Verb-noun pattern works well.
2. **Description**: If everything else were lost, would this explain *why* this matters — without referring to outside documents?
3. **Design**: Is this *how* I'll solve it, kept separate from *what* I'm solving? (Often empty at create time, filled via `br update` as the approach crystallizes.)
4. **Acceptance**: Could a verifier confirm completion from these criteria alone?
5. **Structure**: Are parent/dep links correct? Is the epic-vs-task choice right?

If any answer is "no" or "I'm not sure" — fix it before moving on, or note the gap in the issue's design field for follow-up.

## Supporting files

- **[field-semantics.md](field-semantics.md)** — Each field in depth: what goes in, what doesn't, examples, notes-vs-comments, banned phrases.
- **[structure.md](structure.md)** — Epic decomposition, `--parent` hierarchical IDs, `--deps` types, `--slug` mechanics, external references.
- **[bulk-import.md](bulk-import.md)** — `br create -f` markdown grammar: H2/H3 structure, intra-file references, dependency syntax, worked examples.
- **[agent-context.md](agent-context.md)** — Embedding governing instructions on epics so they surface when descendants are claimed.
