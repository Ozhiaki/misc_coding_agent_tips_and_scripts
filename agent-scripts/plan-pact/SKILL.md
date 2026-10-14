---
name: plan-pact
description: Canonical cross-agent protocol for plan documents (storage, naming, YAML schema, dispute tracking, and archive-after-commit workflow).
---

# Plan Pact Skill

## Purpose

`plan-pact` is a negotiation harness for reasoning agents (human or software) to create, review, dispute, and close planning artifacts without filename drift, location drift, or process thrash.

Use the Decision Register as the current negotiated position and the Decision Log as negotiation history.

This skill is fully self-contained. Treat this file as the canonical protocol.

## Activation Policy

- Starting a new plan-pact cycle requires explicit end-user invocation.
- If planning intent is obvious but invocation is absent, ask for explicit activation confirmation before using plan-pact.
- Exception: when the user asks to update or continue an identified, already-active `.plan.md`, proceed without an extra invocation prompt. A discovered plan is not an authorized update target merely because it has the same project, topic, or file type.

## Progressive Disclosure

Read only what you need:

- Creating a new plan/ADR/critique: read `TL;DR`, `Storage + Naming`, `YAML Schema`, `Required Sections by Type`, `Permitted Sources`, `Complete Specifications`, `Skill Use`, `Workflow`.
- Updating an existing plan: read `TL;DR`, `Permitted Sources`, `Complete Specifications`, `Skill Use`, `Workflow`, `Review Protocol`, `Dispute Tracking`.
- Handling disagreements: read `Dispute Tracking`.
- Closing work and archiving: read `Archive-After-Commit`.

## Non-Regression Guardrails

Do not degrade these cycle-1 strengths:

- Decision Register + append-only Decision Log.
- Required YAML frontmatter.
- Canonical storage and naming.
- Beads boundary.

## Scope Boundary

- Estimate calculation mechanics are out of scope for plan-pact.
- If effort/time computation is needed, defer that to downstream issue-tracking or implementation tooling.

## TL;DR

1. Active docs go in `plans/`; archived docs go in `plans/archive/`.
2. Use dual extension: `plans/YYYY-MM-DD-<topic-slug>.<type>.md`.
3. Allowed types: `plan`, `adr`, `critique`.
4. `spec` is an alias of `plan` (canonical type/filename remains `.plan.md`).
5. YAML frontmatter is required.
6. Edit in place only when the user asks to update or continue that identified plan.
7. `.plan.md` requires `Inspiration`, `Decision Register`, and append-only `Decision Log`.
8. `OPEN` is default; if `Flip Count >= 2`, set `DISPUTED`; only human resolution can set `LOCKED`.
9. Decision Log entries require actor, decision ID, change kind, and rationale.
10. The actor/tool that makes implementation commit must archive the plan file when appropriate.
11. Use permitted sources only; make the specification complete without its authoring sources; refer to governing skills by name.

## Storage + Naming

Canonical storage:
- Active artifacts: `plans/`
- Archived artifacts: `plans/archive/`
- Do not store canonical plan artifacts in repo root.
- `history/` is non-canonical session notes.

Filename pattern:
- `plans/YYYY-MM-DD-<topic-slug>.<type>.md`

Examples:
- `plans/2026-02-16-board-creation-ux.plan.md`
- `plans/2026-02-16-header-ownership.adr.md`
- `plans/2026-02-16-board-creation-ux.critique.md`

Naming rules:
- `<topic-slug>` is lowercase kebab-case.
- Do not include agent names in filenames.
- Do not use `-v2`/`-v3`; use supersession fields instead.
- If a new draft's canonical filename already exists, choose a distinct, descriptive topic slug and use it in both the filename and frontmatter. Do not overwrite the existing plan or select it as the update target. Use supersession only for an authorized replacement.

## YAML Schema (Required)

Every plan artifact must begin with YAML frontmatter:

```yaml
---
title: Board Creation UX
type: plan
date: 2026-02-16
topic: board-creation-ux
status: draft
authors:
  - codex
project: agwakwagan
supersedes: null
superseded_by: null
reviewed_by: []
---
```

Required keys:
- `title` (string)
- `type` (`plan` | `adr` | `critique`; incoming `spec` normalizes to `plan`)
- `date` (`YYYY-MM-DD`)
- `topic` (kebab-case string)
- `status` (`draft` | `active` | `blocked` | `complete` | `superseded`)
- `authors` (non-empty list of strings)
- `project` (string)

Optional keys:
- `supersedes` (string path or `null`)
- `superseded_by` (string path or `null`)
- `reviewed_by` (list of strings)

## Required Sections by Type

`.plan.md` must contain:
- Inspiration
- Problem Summary
- Goals
- Decision / Approach
- Implementation Plan
- Acceptance Criteria
- Open Questions
- Decision Register
- Decision Log (append-only)

`.adr.md` must contain:
- Context
- Decision
- Rationale
- Consequences

`.critique.md` must contain:
- Findings
- Risks
- Recommendations

## Permitted Sources

- Before drafting, identify the source documents permitted by the user's request. Use the named documents and companion documents that those sources explicitly incorporate. Other documents found in the workspace are outside the source set; their presence does not authorize their use.
- Do not use an earlier plan to supply requirements, decisions, structure, or history unless the user includes it in the source set or asks to continue it. If a necessary source choice is unclear, ask about that choice. Finding an unrelated plan is not, by itself, a reason to stop and ask.
- A request to draft a specification from a brief is a drafting request. Edit a plan in place only when the user asks to update or continue that identified plan.
- For a new specification, create metadata, the Decision Register, and the Decision Log from permitted sources and current drafting work. Do not copy authors, dates, decision IDs, dispute states, or log entries from an excluded plan. Preserve append-only history only when that plan is the authorized update target.
- Identify decisions from permitted sources as `Source decision` and new choices as `Proposal` in the register's `Current Choice` cells and the log rationale. These labels identify origin; they do not establish user approval. Keep the existing meanings of `OPEN`, `DISPUTED`, and `LOCKED`.

## Complete Specifications

- A plan specification must contain the information needed to create implementation tasks without reading its source documents. Treat briefs, notes, and discussions as authoring inputs.
- Include all applicable requirements, decisions, dependencies, and acceptance criteria in the plan. Do not use a source document to supply a missing requirement.
- In `Inspiration`, name permitted source documents and their versions in plain text. Do not include hyperlinks, filesystem paths, or instructions to read those documents.
- State unresolved questions in the plan. Do not direct the reader elsewhere to find them.
- Before delivery, read the plan with its source documents unavailable. Can an agent create implementation tasks and acceptance checks from the plan alone? Add missing required information and remove links and reading instructions that lead to authoring sources. Later agents may still use explicitly named governing skills.

## Skill Use

- Read and apply authorized governing skills while drafting. Apply their guidance to the product's stated scope. Do not turn every skill topic into a feature or deliverable.
- Refer to governing skills by name. Each agent must resolve them through its own environment and conventions. Do not put skill installation paths, local hyperlinks, or internal reference-file paths in the plan. This restriction does not prohibit filenames for required product artifacts or implementation files.
- Include the resulting product requirements, decisions, and acceptance criteria. Do not include skill file lists, reading sequences, or accounts of how the author applied a skill.
- Where later work must use a skill, give one instruction by name, such as: "Apply make-cli when planning, reviewing, or implementing CLI work." Do not repeat internal navigation instructions or create a task to copy skill documents into the repository unless the user requires it.
- For each new product choice derived from a skill, state the relevant need and label it as a proposal if permitted sources do not decide it. Define required product contract stability directly; do not substitute skill-copying or skill-version rules for product requirements.

## Workflow

### Create New Artifact

1. Choose type: `plan`, `adr`, or `critique`.
2. Choose topic slug.
3. Create a file with a canonical name in `plans/`; resolve filename conflicts as specified in `Storage + Naming`.
4. Add required YAML frontmatter.
5. Fill required body sections.
6. For `plan`, create the `Decision Register` and `Decision Log` from permitted sources and current drafting work. Label source decisions and proposals as specified in `Permitted Sources`. Record each initial choice with a `confirmed` log entry using the current author and date; this records the drafting work and does not imply user approval.
7. Perform the source independence check in `Complete Specifications` before delivery.

### Update Existing Active Plan

1. Edit the identified active `.plan.md` in place only within the user's authorized update task. Preserve its append-only history.
2. Update frontmatter `status` as appropriate (`draft`, `active`, `blocked`).
3. When changing tracked decisions, update register and append a log entry.

### Replace/Supersede Plan

Use only for an authorized replacement with a materially different direction:

1. Create new plan file in `plans/`.
2. Set new file `supersedes` to old file path.
3. Update old file:
   - `status: superseded`
   - `superseded_by: <new-file-path>`
4. Keep both files until archive step.

### Critique Another Plan

1. Create `.critique.md` in `plans/`.
2. Record target file in body and/or `reviewed_by`.
3. Keep critique separate from the target plan file.

## Review Protocol

Reviewers should use this order:

1. Permitted sources: confirm that metadata, requirements, tasks, and decision history use permitted sources and contain no content inherited from excluded plans. Confirm that source decisions and proposals are identified separately.
2. Skill use: confirm that governing skills are authorized and referenced by name only. Identify installation paths, internal file lists, and author reading records for removal. Confirm that skill guidance has not created unexplained product scope.
3. Complete specifications: perform the source independence check. The specification must supply required information without directing readers back to authoring sources. Later agents may still use explicitly named governing skills.
4. Decision Register and Decision Log consistency.
5. Decision-to-task alignment.
6. Scope creep and non-goal violations.
7. Residual risk and accessibility implications.

Required review output format:

1. Numbered findings.
2. Each finding tagged as `substantive`, `cosmetic`, or `question`.

Submission rule:

- Reviewers submit findings.
- Report failures of the source and skill checks as findings; corrections follow the same author/approval rule as other review edits.
- Edits are applied only by the plan author or after explicit human approval.

## Dispute Tracking

All `.plan.md` files must include:

Decision Register columns:
- `ID` (stable identifier, e.g. `SPEC-012`)
- `Current Choice`
- `Flip Count`
- `Status` (`OPEN` | `DISPUTED` | `LOCKED`)
- `Last Editor`
- `Last Updated`

Decision status semantics:
- `OPEN`: default working state; autonomous edits are allowed.
- `DISPUTED`: flip threshold reached; autonomous edits are frozen pending human decision.
- `LOCKED`: set only after human resolution of a disputed decision.

Decision Log format (append-only):
- `changed|removed|superseded`: `YYYY-MM-DD <editor> <ID>: <change-kind> <old> -> <new> (reason: <reason>)`
- `confirmed`: `YYYY-MM-DD <editor> <ID>: confirmed <choice> (reason: <reason>)`

Allowed `change-kind` values:
- `changed`
- `confirmed`
- `removed`
- `superseded`

Rules:
1. A flip occurs when a decision returns to a prior choice (`A -> B -> A`).
2. Any change to a tracked decision appends a Decision Log entry.
3. If `Flip Count >= 2`, set `Status = DISPUTED` and freeze autonomous edits for that decision.
4. `DISPUTED` requires human resolution.
5. After human resolution, set `Status = LOCKED`.
6. No direct `OPEN -> LOCKED` transition is allowed.
7. `LOCKED` decisions reopen only with new evidence or changed requirements.
8. No-op changes are invalid (`A -> A` is not allowed).
9. Decision removals/merges require explicit rationale in the log entry.

## Archive-After-Commit

The actor/tool performing implementation commit is responsible for archive movement:

1. Complete implementation commit(s).
2. Move finished/superseded plan files from `plans/` to `plans/archive/`.
3. Update frontmatter status/links (`complete` or `superseded`, `supersedes`, `superseded_by`).
4. Commit the archive move as follow-up commit by the same actor/tool.

## Handoff Block

When handing off between agents, include:
- `Active Plan: <absolute-path>`
- `Status: <frontmatter-status>`
- `Next Decision Needed: <short text>`

## Beads Boundary

- Plan artifacts do not include beads commands or issue IDs unless explicitly requested.
- Convert approved plan outcomes into beads work in a separate step.

## Templates

- `templates/plan-template.plan.md`
- `templates/adr-template.adr.md`
- `templates/critique-template.critique.md`
