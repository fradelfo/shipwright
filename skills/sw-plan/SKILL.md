---
name: sw-plan
description: >
  Use when transitioning from discovery to implementation, starting a
  standard-tier feature, or when the user needs a structured implementation
  plan before coding. Also use when the user says "plan the implementation",
  "how should I build this", or "break this into tasks."
argument-hint: [feature to plan, or path to discovery doc]
allowed-tools: Read, Write, Glob, Grep, Bash(mkdir *), Bash(ls *)
---

# Shipwright Plan

Architect-led implementation design: codebase analysis, architecture decisions with ADRs, ordered task sequence, and verification strategy.

## Role Loading

Read and apply this role perspective for this phase:

- [Software Architect](../roles/software-architect.md) (lead role)

Note: Developer role arrives in P2. Until then, the Architect covers implementation sequencing.

If the role file cannot be loaded: "WARNING: Could not load Software Architect role definition. Proceeding with reduced capability."

## Context Bootstrapping

If `$ARGUMENTS` is a file path, read it as upstream context (typically a discovery doc from `sw-discover`). Extract:

- Problem statement and success criteria
- Scope decisions (in/out list)
- Feasibility notes and known risks

If `$ARGUMENTS` is a text description, proceed without upstream — the "never blocks" principle applies. Note the absence of a discovery doc and proceed with what the user has provided.

## Outcome

Produce an implementation plan containing ALL of these sections:

- **Codebase context** — existing patterns and conventions to follow
- **Architecture decisions** with inline ADRs for non-obvious choices
- **Implementation sequence** — ordered tasks with file paths
- **Verification strategy** — what test proves each task works
- **Risk flags** — what could go wrong

## How to Get There

Apply the Software Architect perspective. The four activities are guidance, not an enforced sequence:

### 1. Codebase Analysis

Read existing patterns, conventions, and relevant files before proposing anything.

For projects with existing code:
- Identify the directory structure and file naming conventions
- Read key files that the implementation will extend or modify
- Note the technology stack, frameworks, and libraries in use
- Identify coding style (test framework, linting conventions, etc.)

For greenfield projects:
- Note the absence of existing patterns explicitly
- Ask the user for preferences on technology choices, directory structure, and coding conventions
- All choices are non-obvious when there are no existing patterns — be explicit about each one

Output: Pattern inventory and conventions list.

### 2. Architecture Decisions

Design the components, data flows, and interfaces needed to implement the feature.

For any non-obvious architectural choice, write an inline ADR using the format from the [Software Architect role file](../roles/software-architect.md):

```
## ADR: <title>
**Status:** Accepted
**Context:** What situation are we in?
**Options:** What did we consider?
**Decision:** What did we pick and why?
**Consequences:** What follows from this?
```

What counts as non-obvious: any choice where a reasonable developer might pick a different option. If there's only one sensible approach, don't write an ADR — just do it.

Output: Component diagram (Mermaid if helpful), interface contracts, ADRs for non-trivial decisions.

### 3. Implementation Sequence

Break the implementation into ordered, independently testable steps.

Each step must specify:
- What to build (one sentence)
- Which files to create or modify
- What the step depends on (if anything)

Prefer a sequence where each step produces something runnable or testable. Don't plan a sequence where everything comes together only at the end.

Output: Numbered task list with file paths.

### 4. Verification Strategy

For each step in the implementation sequence, define:
- What test or check proves it works?
- What could go wrong?

Verification can be a unit test, an integration test, a manual check, or a shell command — whatever is appropriate for the step.

Output: Verification criteria per step.

## Save Artifact

Before writing, run:

```bash
mkdir -p docs/shipwright/plan/
```

Save the plan document to `docs/shipwright/plan/YYYY-MM-DD-<topic>.md`.

Match the `<topic>` slug to the discovery doc's topic if one was provided. Otherwise, derive a concise kebab-case slug from the feature description.

Use this frontmatter in the artifact:

```yaml
---
type: plan
topic: <topic-slug>
tier: <major|standard>
status: complete
date: YYYY-MM-DD
upstream: <path-to-discovery-doc, or null>
---
```

Write the artifact to disk **before** presenting transition options.

## Transition

After saving, announce:

> "Plan ready at `docs/shipwright/plan/YYYY-MM-DD-<topic>.md`. Start building with your preferred approach."

The Build skill (`sw-build`) arrives in P2. The plan document is immediately useful as a standalone artifact — it can guide manual implementation or any build tool.

## Plan Template

```markdown
---
type: plan
topic: <topic-slug>
tier: standard
status: complete
date: YYYY-MM-DD
upstream: null
---

# Plan: <Feature Name>

## Overview
[One paragraph: what we're building and why, from the discovery doc or user description.]

## Codebase Context
[Existing patterns, conventions, and relevant files. For greenfield: technology choices made.]

## Architecture Decisions

[ADR blocks for non-obvious choices. Omit if all choices are obvious.]

## ADR: <title>
**Status:** Accepted
**Context:** [Situation]
**Options:** [What was considered]
**Decision:** [What was chosen and why]
**Consequences:** [What follows]

## Implementation Sequence

1. **[Task name]** — [One-sentence description]
   - Files: `path/to/file.ext` (create), `path/to/other.ext` (modify)
   - Depends on: [task number, or "nothing"]

2. **[Task name]** — [One-sentence description]
   - Files: `path/to/file.ext` (create)
   - Depends on: task 1

## Verification Strategy

| Task | What proves it works | What could go wrong |
|------|---------------------|---------------------|
| 1    | [Test or check]     | [Risk]              |
| 2    | [Test or check]     | [Risk]              |

## Risk Flags

- [Risk 1 and mitigation]
- [Risk 2 and mitigation]
```

## Principles

- Read the codebase before proposing architecture. Understand what exists.
- Suggest, never block. If the user wants to skip planning and build directly, let them.
- Prefer boring technology. The simplest design that handles requirements wins.
- Every task in the implementation sequence must have a verification criterion.
- ADRs are lightweight — four fields, not a ceremony. Write one only when a choice is genuinely non-obvious.
- For greenfield projects, every technology choice is non-obvious. Make them explicit.
