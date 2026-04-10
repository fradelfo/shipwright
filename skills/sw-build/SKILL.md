---
name: sw-build
description: >
  Use when ready to implement a plan, when the user says "let's build this",
  "start coding", "implement the plan", or "write the code". Also use when
  transitioning from sw-plan and an implementation plan artifact is available,
  or when the user wants to implement a feature without a formal planning phase.
argument-hint: [path to plan artifact, or feature description]
allowed-tools: Read, Write, Glob, Grep, Bash(*)
---

# Shipwright Build

Developer-led implementation: read the plan, run a TDD loop per task, QA companion after each, write a build artifact.

Note on `allowed-tools`: `Bash(*)` is an intentional exception to the principle of least privilege. Test commands are project-specific (`npm test`, `pytest`, `go test`, `bundle exec rspec`, etc.) and cannot be enumerated ahead of time. The build phase requires broad shell access.

## Role Loading

Read and apply these role perspectives for this phase:

- [Developer](../roles/developer.md) (lead role)
- [QA](../roles/qa.md) (continuous companion)

If a role file cannot be loaded: "WARNING: Could not load [resource]. Proceeding with reduced capability."

## Context Bootstrapping

If `$ARGUMENTS` is a file path ending in `.md`, read it as an upstream plan artifact. Extract:

- Task list with verification criteria
- File paths to create or modify
- Architecture decisions and ADRs
- Risk flags

If `$ARGUMENTS` is a text description, synthesise an ad-hoc task sequence. Note the absence of a formal plan artifact and proceed.

If `$ARGUMENTS` is empty, ask: "What should I build? Provide a plan artifact path or describe the feature."

## Outcome

Produce:

- **Working code** — all tasks from the plan implemented and verified
- **Build artifact** — `docs/shipwright/build/YYYY-MM-DD-<topic>.md` documenting task completion status, modified files, and QA findings

Both are required before the transition to sw-ship.

## How to Get There

Apply the Developer perspective as lead. QA activates after each task completion. The four activities below are guidance, not an enforced sequence.

### 1. Plan Ingestion

Read the plan artifact (or synthesise tasks from the description). Extract:

- Ordered task list with file paths
- Verification criterion per task
- Dependencies between tasks
- Risk flags noted by the Architect

Confirm task count with the user: "Found N tasks. Starting with task 1: [description]. Proceed?"

### 2. TDD Loop (per task)

For each task in order:

1. **State assumptions** — one sentence per assumption before touching code
2. **Write the test first** — or state the manual verification step if no test framework applies
3. **Read before modifying** — read all files the task will touch before changing them
4. **Implement** — surgical changes only; match existing style; touch nothing outside the task's file scope
5. **Verify** — run the verification criterion from the plan; confirm it passes
6. **QA companion check** — hand off to QA role; address Critical and High findings before proceeding
7. **Mark task complete** — update the build artifact

**Verification failure:** If the verification criterion fails after implementation, stop. Surface the failure to the user with the failing test output. Do not retry automatically — the user decides whether to fix or skip.

### 3. QA Companion Activation

After each task completes, the QA role asks its core questions against the just-implemented code:

- Empty/null/enormous input handled?
- Called twice — idempotent?
- Concurrent call — safe?
- Error path — what does the caller see?
- Most expensive assumption — named?

QA rates any findings Critical / High / Medium / Low. Critical and High findings are addressed before moving to the next task. Medium and Low are advisory — recorded in the build artifact.

**Quick tier:** If the tier is Quick and no QA companion is warranted, the Developer role runs its self-QA checklist instead (defined in the Developer role file).

### 4. Write Build Artifact

Before presenting the transition:

```bash
mkdir -p docs/shipwright/build/
```

Write `docs/shipwright/build/YYYY-MM-DD-<topic>.md` using the template below. Write the artifact to disk before announcing the transition.

## Save Artifact

Save to `docs/shipwright/build/YYYY-MM-DD-<topic>.md`.

Match the `<topic>` slug to the upstream plan artifact's topic if one was provided.

Use this frontmatter:

```yaml
---
type: build
topic: <topic-slug>
tier: <quick|standard|major>
status: complete
date: YYYY-MM-DD
upstream: <path-to-plan-artifact, or null>
---
```

## Transition

After saving the build artifact, announce:

> "Build complete. Invoke `/sw-ship docs/shipwright/build/YYYY-MM-DD-<topic>.md` to review and ship."

Pass the explicit artifact path — do not rely on fuzzy matching.

## Build Artifact Template

```markdown
---
type: build
topic: <topic-slug>
tier: standard
status: complete
date: YYYY-MM-DD
upstream: null
---

# Build: <Topic>

## Tasks

| # | Description | Status | Notes |
|---|-------------|--------|-------|
| 1 | [task description] | ✓ pass | |
| 2 | [task description] | ✓ pass | QA: Medium finding noted |
| 3 | [task description] | ✗ fail | Skipped by user — [reason] |

## Modified Files

- `path/to/file.ext`
- `path/to/other.ext`

## QA Findings (Build Phase)

| Severity | Finding | Addressed |
|----------|---------|-----------|
| High | [description] | Yes — [how] |
| Medium | [description] | Deferred — noted for ship review |
| Low | [description] | Advisory |
```

## Principles

- Read before modifying. Never implement from memory.
- Surgical changes — touch only what the task requires.
- Every task must have a verification criterion before implementation begins.
- Declare done only when the verification criterion passes, not when the code looks right.
- QA companion is a collaborator, not a gatekeeper. Critical and High findings block; Low findings do not.
- Write the build artifact before presenting the transition — never lose progress.
- Suggest, never block. If the user wants to skip a task or override a QA finding, let them.
