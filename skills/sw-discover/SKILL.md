---
name: sw-discover
description: >
  Use when starting a major feature, when requirements are unclear, when
  the user needs help defining what to build, or when scope is vague or
  ambiguous. Also use when the user says "let's think about this", "I'm
  not sure what to build", or "help me figure out requirements."
argument-hint: [feature or problem to discover]
allowed-tools: Read, Write, Glob, Bash(mkdir *)
---

# Shipwright Discovery

PM-led exploration of what to build and why.

## Role Loading

Read and apply these role perspectives for this phase:

- [Product Manager](../roles/product-manager.md) (lead role)
- [UX Designer](../roles/ux-designer.md) (support role)
- [Software Architect](../roles/software-architect.md) (feasibility check)

## Context

If `$ARGUMENTS` is a file path, read it as context. It could be an existing discovery doc to iterate on, or upstream requirements from another source.

Otherwise, treat `$ARGUMENTS` as the feature or problem description and use it as the starting point for discovery.

## Outcome

Produce a discovery document containing ALL of these sections:

- **Problem statement and success criteria** — What problem, for whom, measured how.
- **User flows** — Happy path and error paths, described step by step.
- **Feasibility assessment** — Viability, blockers, and unknowns from an architecture perspective.
- **Scoped MVP feature list** — Explicit in/out columns with rationale for each exclusion.

## How to Get There

Load the PM, UX, and Architect role perspectives. Use judgment about which perspective to apply first based on the user's input:

- If the problem itself is vague, start with PM (problem framing).
- If the problem is clear but flows are vague, start with UX (user flow mapping).
- If the problem and flows are clear but technical feasibility is unknown, start with Architect.

The four activities below are guidance, not an enforced sequence. Adapt to the user's needs.

### 1. Problem Framing (PM perspective)

Apply the Product Manager core questions. Ask clarifying questions one at a time — do not dump all questions at once.

Output:
- Problem statement (one to three sentences)
- Success criteria (measurable, testable)
- Explicit non-goals

### 2. User Flow Mapping (UX perspective)

Apply the UX Designer core questions. Map the happy path first, then error and edge-case paths.

Output:
- User flows (numbered steps in plain text; use Mermaid only when the flow has complex branching)
- Screen inventory (purpose, key elements, entry/exit points)

### 3. Feasibility Gut-Check (Architect perspective)

Apply the Software Architect core questions. Read existing codebase patterns if any code exists — use Glob and Read to survey the project structure.

Output:
- Viability assessment (simple / moderate / significant complexity)
- Obvious blockers
- Big unknowns that need resolution before or during build

### 4. Scope Negotiation (PM perspective)

Given the flows from UX and the feasibility assessment from the Architect, define the MVP. Cut aggressively.

Output:
- Scoped feature list with explicit **In** and **Out** columns
- Phase 2 candidates (things cut from MVP that should come back later)

## Save Artifact

Save the discovery document to `docs/shipwright/discover/YYYY-MM-DD-<topic>.md`.

Before writing:

- Run `mkdir -p docs/shipwright/discover/`
- Derive the `<topic>` slug as a concise kebab-case description (e.g., `payment-system`, `user-onboarding`)
- Replace `YYYY-MM-DD` with the current date

Include this YAML frontmatter in the artifact:

```yaml
---
type: discover
topic: <topic-slug>
tier: major
status: complete
date: YYYY-MM-DD
upstream: null
---
```

Write the artifact to disk BEFORE presenting transition options.

## Transition

After saving, announce:

> Discovery complete. Invoke `/sw-plan docs/shipwright/discover/YYYY-MM-DD-<topic>.md` to plan the implementation.

Pass the explicit artifact path — do not rely on fuzzy matching.

The user can also:

- Skip to build directly
- Refine the discovery doc further
- Stop here

## Discovery Template

The output document must follow this structure:

```markdown
---
type: discover
topic: <topic-slug>
tier: major
status: complete
date: YYYY-MM-DD
upstream: null
---

# Discovery: <Topic Title>

## Problem Statement

[One to three sentences. State the problem, who has it, and why it matters.]

## Success Criteria

- [Measurable outcome 1]
- [Measurable outcome 2]
- [Measurable outcome 3]

## Non-Goals

- [Explicit exclusion 1 — reason]
- [Explicit exclusion 2 — reason]

## User Flows

### Happy Path

1. [Step 1]
2. [Step 2]
3. [Step 3]

### Error Paths

1. [Error scenario — what happens, how the user recovers]

## Screen Inventory

| Screen | Purpose | Entry | Exit |
|--------|---------|-------|------|
| [Name] | [What the user does here] | [How they arrive] | [Where they go] |

## Feasibility Assessment

**Complexity:** [simple / moderate / significant]

**Blockers:**
- [Blocker or "None identified"]

**Unknowns:**
- [Unknown or "None identified"]

## Scoped MVP Feature List

| In | Out | Reason for exclusion |
|----|-----|---------------------|
| [Feature] | | |
| | [Deferred feature] | [Why it was cut] |

## Phase 2 Candidates

- [Feature deferred to phase 2 — brief rationale]
```

## Principles

- Suggest, never block. If the user wants to skip steps, let them.
- Each role perspective should be clearly distinct — PM thinks in problems, UX thinks in journeys, Architect thinks in trade-offs.
- The output document should be useful even if the user stops here and never plans.
- Prefer text-based user flows. Use Mermaid only when the flow has complex branching.
- Ask clarifying questions one at a time, not in batches.
- Cut scope aggressively for MVP. Everything cut goes to Phase 2, nothing is lost.
