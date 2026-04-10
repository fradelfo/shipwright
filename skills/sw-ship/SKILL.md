---
name: sw-ship
description: >
  Use when the build is complete and ready for review, when the user says
  "review this", "ship it", "get this ready for a PR", or "run QA". Also
  use when transitioning from sw-build and a build artifact is available,
  or when the user wants a code review on manually written changes.
argument-hint: [path to build artifact, or nothing to derive from git diff]
allowed-tools: Read, Write, Glob, Grep, Bash(git diff *), Bash(git log *), Bash(git status)
---

# Shipwright Ship

QA-led review and release preparation: adversarial testing, parallel review agents, fix loop, changelog and PR description, ship artifact.

## Role Loading

Read and apply these role perspectives for this phase:

- [QA](../roles/qa.md) (lead role — testing and release prep)
- [Developer](../roles/developer.md) (support role — fix loop)

If a role file cannot be loaded: "WARNING: Could not load [resource]. Proceeding with reduced capability."

## Context Bootstrapping

If `$ARGUMENTS` is a file path ending in `.md`, read it as an upstream build artifact. Extract:

- Modified files list
- Task completion status (pass/fail per task)
- QA findings from the build phase
- Upstream plan artifact path (via the `upstream:` frontmatter field)

If `$ARGUMENTS` is empty, derive the review scope from the git diff:

```bash
git diff main...HEAD
```

Note the absence of a build artifact — QA will conduct a full exploratory pass without build-phase context.

**Pre-check:** Run `git status` to verify the working tree is clean (all changes committed). If uncommitted changes are detected, warn the user: "Uncommitted changes found. Commit or stash before running sw-ship to ensure the review covers the complete change set." The user may proceed anyway — never block.

## Outcome

Produce:

- **Ship artifact** — `docs/shipwright/ship/YYYY-MM-DD-<topic>.md` with consolidated review findings, PR title and body, changelog entry, and retrospective
- **CHANGELOG.md append** — Keep a Changelog entry appended to `CHANGELOG.md` at the repo root (if the file exists)
- **gh pr create command** — ready-to-copy command with pre-filled title and body file

## How to Get There

Apply the QA perspective as lead. The five activities below are guidance, not an enforced sequence.

### 1. Scope Definition

Identify what changed:

- Read the build artifact (if provided) for the modified files list and task completion status
- Run `git diff main...HEAD` (or appropriate base branch) to get the full diff
- Note any tasks marked ✗ fail in the build artifact — surface these to the user before testing

### 2. Exploratory Testing (QA lead, adversarial)

Apply the QA role's core questions against the changed code. This is not a scripted checklist — it is an adversarial exploration:

- Try to break it with unexpected inputs
- Trace failure paths end to end
- Look for silent failures, unchecked assumptions, and missing error handling
- Note findings with severity: Critical / High / Medium / Low

### 3. Parallel Review Agent Dispatch

Dispatch four review agents simultaneously, each examining the diff from a distinct perspective:

- **[Correctness](../../agents/review/correctness.md)** — logic errors, edge cases, test coverage gaps
- **[Security](../../agents/review/security.md)** — vulnerabilities, input validation, auth surfaces
- **[Simplicity](../../agents/review/simplicity.md)** — YAGNI violations, over-engineering, unnecessary complexity
- **[Performance](../../agents/review/performance.md)** — N+1 queries, unbounded growth, blocking operations

Input to each agent:
- The git diff (or targeted file contents if the diff is large)
- The plan artifact path (for scope context — what the task was)

**Fallback:** If parallel sub-agent dispatch is not supported in the current execution context, conduct each review perspective sequentially in-conversation using the agent file as the instruction set.

Collect findings from all four agents. Merge by severity — Critical and High findings from any agent require resolution or explicit user override before shipping.

### 4. Fix Loop (if blocking issues found)

For each Critical or High finding:

1. Present the finding with reproduction steps to the user
2. Developer role: implement the fix — surgical change, matching existing style
3. Re-run the relevant review agent on the changed section
4. Confirm the finding is resolved

The user can override: "ship anyway." Never block — record the override in the ship artifact.

### 5. Release Prep (QA release preparation stance)

After testing and review are complete, switch to the release preparation stance (QA role, Release Manager responsibilities).

**Changelog entry** — Write a Keep a Changelog entry:

```markdown
## [Unreleased] — YYYY-MM-DD
### Added
- [user-facing description of new capability]
### Changed
- [user-facing description of change]
### Fixed
- [user-facing description of fix]
```

Append this entry to `CHANGELOG.md` at the repo root if the file exists. Create it if the user asks; do not create it silently.

**PR title** — Conventional commits format: `type(scope): short description`

**PR body** — Structure:

```markdown
## What
[What changed and why — two to four sentences]

## How to Test
[Numbered steps a reviewer can follow to verify the change]

## Deployment Notes
[Env vars, migrations, config changes — or "None"]
```

**Retrospective** — One short paragraph: what went well, what was harder than expected, what to do differently next time.

**gh pr create command** — Assemble and output (do not execute):

```bash
gh pr create \
  --title "<PR title>" \
  --body-file docs/shipwright/ship/YYYY-MM-DD-<topic>.md
```

## Save Artifact

Before presenting the gh pr create command:

```bash
mkdir -p docs/shipwright/ship/
```

Write `docs/shipwright/ship/YYYY-MM-DD-<topic>.md` using the template below. Write the artifact to disk first.

Match the `<topic>` slug to the upstream build artifact's topic if one was provided.

Use this frontmatter:

```yaml
---
type: ship
topic: <topic-slug>
tier: <quick|standard|major>
status: complete
date: YYYY-MM-DD
upstream: <path-to-build-artifact, or null>
---
```

## Transition

After saving the ship artifact, announce:

> "Ready to ship. Run:"
>
> ```bash
> gh pr create \
>   --title "<PR title>" \
>   --body-file docs/shipwright/ship/YYYY-MM-DD-<topic>.md
> ```

## Ship Artifact Template

```markdown
---
type: ship
topic: <topic-slug>
tier: standard
status: complete
date: YYYY-MM-DD
upstream: null
---

# Ship: <Topic>

## Review Findings

### Exploratory Testing

[What was tried, what held up, what broke]

### Agent Findings

| Agent | Severity | Finding | Location | Resolution |
|-------|----------|---------|----------|------------|
| Correctness | High | [description] | [file:line] | Fixed — [how] |
| Security | None | | | No issues found |
| Simplicity | Low | [description] | [file:line] | Advisory |
| Performance | None | | | No issues found |

### Overall Recommendation

ship it / fix first

## PR Title

type(scope): description

## PR Body

## What
[What changed and why]

## How to Test
1. [Step 1]
2. [Step 2]

## Deployment Notes
None

## Changelog Entry

## [Unreleased] — YYYY-MM-DD
### Added
- [user-facing description]

## Retrospective

[What went well / what was harder / what to do differently]
```

## Principles

- Adversarial testing first. Try to break it before praising it.
- Dispatch review agents in parallel when possible — four independent lenses catch more than one sequential sweep.
- Pass scoped input to each agent (the diff and plan path), not the full conversation context.
- Release prep is non-negotiable. Even clean-looking code gets a changelog entry and PR description.
- Generate the gh pr create command; do not execute it. The PR creation decision belongs to the user.
- Write the ship artifact before outputting the command — never lose the release notes.
- Suggest, never block. Critical findings require resolution or explicit user override; they do not hard-block.
