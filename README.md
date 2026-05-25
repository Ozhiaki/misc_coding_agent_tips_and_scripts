# Misc Coding Agent Tips and Scripts

Operational guidance for AI coding agents—not setup docs, but best practices for using tools *well*.

## Philosophy

Most agent documentation answers "how do I install this?" This repo answers "how do I use it thoughtfully?"

An agent can learn commands from `--help` but can't infer *why* certain patterns exist or when to apply them. These docs fill that gap.

## Contents

---

### Operational Guides

Reference docs for agents. Link to these from your `AGENTS.md` or `CLAUDE.md` so agents understand *why* certain patterns exist and when to apply them.

#### [Best Practices For beads-rust](agent-operational-guides/beads-rust/)

Operational guidance for `br` (beads_rust). Covers issue design, field semantics, and agent workflows.

| Document | Purpose |
|----------|---------|
| [README](agent-operational-guides/beads-rust/README.md) | Operational differences, hierarchical IDs, claim workflow |
| [Issue Design](agent-operational-guides/beads-rust/ISSUE_DESIGN.md) | Why fields exist and how to use them |
| [Field Semantics](agent-operational-guides/beads-rust/FIELD_SEMANTICS.md) | Description vs design vs acceptance vs notes |
| [When to Create Issues](agent-operational-guides/beads-rust/WHEN_TO_CREATE_ISSUES.md) | Recognizing and filing discovered work |
| [Issues vs Planning Docs](agent-operational-guides/beads-rust/ISSUES_VS_PLANNING_DOCS.md) | Why issues should be self-contained |
| [Code in Issues](agent-operational-guides/beads-rust/CODE_IN_ISSUES.md) | When to include code snippets |
| [Examples](agent-operational-guides/beads-rust/EXAMPLES.md) | Before/after issue creation patterns |
| [Session Protocol](agent-operational-guides/beads-rust/SESSION_PROTOCOL.md) | Session lifecycle — start, claim, work, close, flush |
| [Agent Integration](agent-operational-guides/beads-rust/AGENT_INTEGRATION.md) | Wiring best practices into agent toolchains (three-layer model) |
| [Safety](agent-operational-guides/beads-rust/SAFETY.md) | Sync guards, history, data protection |

---

### Agent Skills

Copyable skill files for Claude, Codex, and other agents. Drop a skill into your agent's skill directory and it becomes available as a slash command.

#### [plan-to-beads](agent-scripts/plan-to-beads/)

Agent skill for the distillation step: translating a planning document (markdown spec, design doc, requirements list) into well-structured `br` issues — epics, hierarchical children, dependencies, and properly-separated fields. Counterpart to `beads-to-done` (which handles the execution phase); this skill covers issue construction only.

| Document | Purpose |
|----------|---------|
| [SKILL.md](agent-scripts/plan-to-beads/SKILL.md) | Trigger, dispatch, four-field content model summary |
| [field-semantics.md](agent-scripts/plan-to-beads/field-semantics.md) | Each field in depth: description/design/acceptance/notes, examples, anti-patterns, notes-vs-comments |
| [structure.md](agent-scripts/plan-to-beads/structure.md) | Epic decomposition, `--parent` hierarchical IDs, `--deps` types, `--slug` mechanics, external references |
| [bulk-import.md](agent-scripts/plan-to-beads/bulk-import.md) | `br create -f` markdown grammar: H2/H3 structure, intra-file references, dependency syntax |
| [agent-context.md](agent-scripts/plan-to-beads/agent-context.md) | Embedding governing instructions on epics so they surface when descendants are claimed |

Key concepts: the four-field content model (description=why, design=how, acceptance=done-when, notes=progress), self-contained issues (no cross-references to plan docs), epic decomposition with hierarchical child IDs, bulk markdown import for >5 related issues, governing context on epics that survives compaction.

#### [beads-to-done](agent-scripts/beads-to-done/)

Agent skill for the execution phase: carrying an existing `br` issue from claim to close. Counterpart to `plan-to-beads` (which constructs issues); this skill covers everything that happens after issues already exist — claiming, working, recording progress, handling mid-work surprises, closing cleanly, and recovering from sync trouble. Solo-agent and cross-session-by-same-agent only; multi-agent / swarm features are explicitly out of scope.

| Document | Purpose |
|----------|---------|
| [SKILL.md](agent-scripts/beads-to-done/SKILL.md) | Trigger, dispatch, the five-phase execution loop at a glance, cross-refs to `plan-to-beads` |
| [claim-and-work.md](agent-scripts/beads-to-done/claim-and-work.md) | Session start, finding ready work, `--claim` mechanics, retrieving the epic's `agent_context`, read-before-claim sanity check |
| [notes-discipline.md](agent-scripts/beads-to-done/notes-discipline.md) | `--notes` overwrite footgun, read-before-write pattern, `--notes` vs `br comments add`, structuring a resumable progress snapshot |
| [discovery.md](agent-scripts/beads-to-done/discovery.md) | Three branches for mid-work surprises: spawn a new bead, fix the current bead, or stop and ask. Wiring `discovered-from` dependencies |
| [closing-and-sync.md](agent-scripts/beads-to-done/closing-and-sync.md) | Acceptance walkthrough, traceable close reasons, `--suggest-next`, the JSONL/DB sync model, the git half of the close |
| [recovery.md](agent-scripts/beads-to-done/recovery.md) | `sync --status` decision tree, three-way merge, JSONL conflict markers, same-agent stuck claims, `br doctor`, exit-code gotchas |

Key concepts: the JSONL is the audit trail / the DB is the working copy, `--notes` overwrites (read-before-write), the three discovery branches (new bead / current bead wrong / stop and ask), `discovered-from` dependencies for traceability, traceable close reasons (what changed + commit ref), work is not done until `git push` succeeds.

#### [Plan-Pact](agent-scripts/plan-pact/)

Cross-agent negotiation protocol for planning documents. Gives multiple reasoning agents (human or software) a shared format for proposing, reviewing, disputing, and closing plans without drift or rewrite loops.

| Document | Purpose |
|----------|----------|
| [SKILL.md](agent-scripts/plan-pact/SKILL.md) | Canonical protocol: storage, naming, frontmatter, dispute tracking, review protocol, activation policy |
| [Plan Template](agent-scripts/plan-pact/templates/plan-template.plan.md) | Starter template for `.plan.md` files with Inspiration section and decision log examples |
| [ADR Template](agent-scripts/plan-pact/templates/adr-template.adr.md) | Starter template for architecture decision records |
| [Critique Template](agent-scripts/plan-pact/templates/critique-template.critique.md) | Starter template for plan critiques |

Key concepts: Decision Register (current negotiated position), Decision Log (append-only negotiation history), Inspiration section (accountability anchor for every plan cycle), explicit activation policy (human invokes, agents don't self-activate).

#### [Tact](agent-scripts/tact/)

Communication skill for writing candid, collegial feedback without revealing rhetorical stage directions. Useful for PRs, reviews, critiques, corrections, refusals, and other relationship-preserving messages.

| Document | Purpose |
|----------|---------|
| [SKILL.md](agent-scripts/tact/SKILL.md) | Full skill: framing patterns, PR body structure, review comments, corrections, and final rewrite checks |

Key concepts: keep the useful point intact, frame around practical benefit, avoid accusatory diagnosis, and remove any sentence that explains why the message is being made tactful.

---

## Usage

**Operational guides** — reference from your `AGENTS.md` or `CLAUDE.md`:

```markdown
## Issue Tracking

This project uses beads_rust (`br`). Follow the guidance at:
https://raw.githubusercontent.com/Ozhiaki/misc_coding_agent_tips_and_scripts/main/agent-operational-guides/beads-rust/README.md
```

**Agent skills** — copy into your agent's skill directory:

```bash
# plan-to-beads (for Claude)
cp -r agent-scripts/plan-to-beads/ .claude/skills/plan-to-beads/

# beads-to-done (for Claude)
cp -r agent-scripts/beads-to-done/ .claude/skills/beads-to-done/

# plan-pact (for Claude)
cp -r agent-scripts/plan-pact/ .claude/skills/plan-pact/

# tact (for Claude)
cp -r agent-scripts/tact/ .claude/skills/tact/

# For Codex, use .codex/skills/ instead
```

## Related Resources

- [Jeffrey Emanuel's misc_coding_agent_tips_and_scripts](https://github.com/Dicklesworthstone/misc_coding_agent_tips_and_scripts) - Setup/configuration focused
- [beads_rust repo](https://github.com/Dicklesworthstone/beads_rust) - Official br documentation and source

## License

MIT
