---
name: tact
description: Use when writing sensitive feedback, PRs, reviews, critiques, refusals, corrections, recommendations, or relationship-preserving messages where the response should be candid but collegial, avoid accusatory framing, and never reveal rhetorical stage directions such as saying the tone is being softened or made diplomatic.
---

# Tact

Use tact to say the useful thing without exposing the social maneuver behind it.

## Core Rule

Do not reveal the rhetorical strategy. Never say or imply that the text is being softened, made diplomatic, made non-accusatory, or written carefully to preserve the relationship.

Remove stage directions such as:

- "the intent is to..."
- "while keeping the tone..."
- "without being harsh..."
- "to avoid sounding accusatory..."
- "this does not try to criticize..."
- "framed conservatively..."
- "to soften the feedback..."

The reader should see a clear, useful message, not the effort spent making it tactful.

## Default Stance

Write as a thoughtful peer who likes the work and wants to improve it.

Prefer:

- practical benefit over diagnosis
- shared goals over fault
- concrete change over motive analysis
- invitation over verdict
- specific evidence over character judgment
- "this helps..." over "this fixes..."

Keep the point intact. Tact is not evasion; it is clean framing.

## Framing Patterns

Use these frames when they fit:

- "This makes X easier to..."
- "This gives future users a clearer path to..."
- "This adds a small guardrail around..."
- "This documents the expected workflow for..."
- "This keeps the current design, while making X more explicit."
- "This should help when..."
- "One small follow-up that may help..."

Avoid frames that expose critique:

- "This corrects the misleading claim that..."
- "This closes the gap between promise and implementation..."
- "This avoids overclaiming..."
- "This makes the project more honest..."
- "This prevents maintainers from assuming..."

## PR Bodies

For a maintainer-friendly PR body, use:

```markdown
## Summary

[One short paragraph describing the practical improvement.]

## Changes

- [Concrete change]
- [Concrete change]
- [Concrete change]

## Validation

- `[command]`
- `[command]`
```

Do not include private review findings, hidden motivation, or social rationale unless the user explicitly asks.

## Review Comments

When writing review comments:

1. Lead with the observable issue.
2. Explain the user or maintainer impact.
3. Suggest a concrete adjustment.
4. Skip speculation about why the issue happened.

Prefer:

```text
This path can leave the request in a partially applied state if the second write fails. Could we wrap both writes in the existing transaction helper?
```

Avoid:

```text
This is unsafe and looks like the implementation forgot about partial failures.
```

## Corrections And Disagreements

When correcting someone:

- acknowledge the useful part if there is one
- state the correction directly
- give evidence or a reproducer
- offer the smallest next step

Avoid performing humility. Do not write "I may be wrong" unless uncertainty is real.

## Rewrite Check

Before finalizing, remove any sentence that explains why the message is gentle. If the sentence is about the communication strategy instead of the subject, delete or rewrite it.

