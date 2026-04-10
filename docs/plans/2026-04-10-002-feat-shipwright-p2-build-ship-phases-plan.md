---
title: "feat: Shipwright P2 — Build and Ship phases"
type: feat
status: completed
date: 2026-04-10
origin: docs/design.md
---

# feat: Shipwright P2 — Build and Ship Phases

## Overview

P1 shipped the plugin foundation: orchestrator, discovery, planning, three role files, and the full methodology skeleton. P2 completes the active lifecycle by adding the two remaining phase skills (`sw-build` and `sw-ship`), two role files (`developer.md`, `qa.md`), and four review agents (`correctness`, `security`, `simplicity`, `performance`). It also updates three existing files to reflect the now-complete skill chain.

After P2, the full Shipwright loop — Discover → Plan → Build → Ship — is functional for Major and Standard tier tasks. Quick tier build remains a direct in-conversation flow.

Origin: [docs/design.md](../../docs/design.md) — Section 3 (Phase descriptions), Section 4 (Roles), Section 5 (Project structure).

---

## Technical Approach

### Architecture

Shipwright is a pure-markdown Claude Code plugin. There is no runtime, no build tooling, no package manager. Every deliverable is a `.md` file loaded by Claude Code at invocation time.

P2 adds two tiers of new files:

**Phase skills** (`skills/sw-*/SKILL.md`) — active slash commands. Claude Code auto-discovers them. Claude reads the full SKILL.md body at invocation.

**Agent files** (`agents/review/*.md`) — sub-agent instruction sets dispatched by sw-ship. Not slash commands; loaded by sw-ship via Task/Agent dispatch. Each file is a focused instruction set for one review lens.

**Role files** (`skills/roles/*.md`) — passive persona definitions. Loaded by skills via markdown links. Not invokable.

### Data flow: Plan → Build → Ship

```
docs/shipwright/plan/YYYY-MM-DD-<topic>.md
        │
        ▼
sw-build reads plan, implements tasks, writes:
docs/shipwright/build/YYYY-MM-DD-<topic>.md
        │
        ▼
sw-ship reads build artifact + git diff, dispatches 4 review agents, writes:
docs/shipwright/ship/YYYY-MM-DD-<topic>.md
+ appends to CHANGELOG.md (if present)
+ outputs gh pr create command
```

Each artifact's `upstream:` frontmatter field links to the previous phase, enabling reliable chain traversal without fuzzy filename matching.

---

## Architecture Decisions

### ADR: Build artifact — write to disk vs. stateless

**Status:** Accepted
**Context:** The design spec says the build phase output is "working code with tests passing" — no document specified. But sw-ship needs to know what was built (which tasks completed, which files changed) to scope its review correctly. Git diff alone is ambiguous: it includes unrelated changes and gives no signal about which plan tasks are done.
**Options:** (1) No build artifact — sw-ship reconstructs from git diff. (2) Build artifact at `docs/shipwright/build/` with task completion status and modified file list.
**Decision:** Write the build artifact. Schema: frontmatter (`type: build`, `topic`, `tier`, `status`, `date`, `upstream: <plan-path>`), plus three sections: completed tasks (pass/fail per verification criterion), modified files list, issues noted during build (QA companion findings).
**Consequences:** sw-ship has a reliable upstream reference. Session resumption becomes possible in a future P3 enhancement. Adds ~10 lines of artifact-writing instruction to sw-build's SKILL.md.

---

### ADR: Review agent invocation — parallel sub-agents vs. sequential in-conversation

**Status:** Accepted
**Context:** The design spec says "spawn parallel review agents if warranted." P1 established that skills cannot programmatically invoke other skills — transitions are conversational. However, Claude Code does support sub-agent dispatch via the Agent/Task tool. The four review lenses (correctness, security, simplicity, performance) are genuinely independent and benefit from parallel execution.
**Options:** (1) Sequential in-conversation persona shifts — simpler, no sub-agent overhead. (2) Parallel sub-agent dispatch from sw-ship — matches design spec, leverages Claude Code native capability.
**Decision:** Parallel sub-agents. sw-ship SKILL.md instructs Claude to dispatch four parallel sub-agents from `agents/review/`, each receiving the git diff and the plan artifact path as context. Each returns a severity-labeled findings block.
**Consequences:** Four agent files needed. Sub-agent dispatch is an established Claude Code pattern (compound-engineering uses it). Each agent receives scoped context (diff + plan) not full conversation — mitigates the context budget risk identified in P1 planning.

---

### ADR: sw-build allowed-tools — specific patterns vs. Bash(*)

**Status:** Accepted
**Context:** The AGENTS.md principle of least privilege requires skills to enumerate allowed tools. sw-plan uses `Read, Write, Glob, Grep, Bash(mkdir *), Bash(ls *)`. sw-build must run tests, which vary by project stack: `npm test`, `pytest`, `go test ./...`, `bundle exec rspec`, etc.
**Options:** (1) Enumerate common test runner patterns — fragile, incomplete. (2) `Bash(*)` — open-ended, violates least privilege principle. (3) Defer test execution to user — skill describes what to run, user executes.
**Decision:** `Bash(*)` for sw-build, as a documented intentional exception. Rationale: the build phase is inherently project-specific; enumerating test runners would be false precision. The exception is recorded in AGENTS.md so future developers understand why.
**Consequences:** Broader Bash access in sw-build. Mitigated by the fact that sw-build is explicitly an implementation phase — broad tool access is expected. Document in AGENTS.md skill compliance checklist as a named exception pattern.

---

### ADR: Review agent file format — skill vs. agent vs. instruction file

**Status:** Accepted
**Context:** P1 established two file types: skills (SKILL.md with `name`, `description`, `allowed-tools` frontmatter) and role files (plain markdown with `name`, `type: role`, `voice`, `phases` frontmatter). Review agents are a third category: not slash commands, not persona definitions — they are sub-agent instruction sets dispatched by sw-ship.
**Options:** (1) Treat as mini-skills with SKILL.md format — would create false `/sw-review-correctness` slash commands. (2) Plain markdown with no frontmatter — loses machine-readability. (3) New frontmatter schema: `name`, `type: agent`, `perspective`, `input`, `output-format`.
**Decision:** New frontmatter schema (option 3). Fields: `name` (matches filename slug), `type: agent`, `perspective` (one word: correctness/security/simplicity/performance), `severity-levels: [critical, high, medium, low]`. Body: one-paragraph focus statement, heuristics checklist, output template. No `description` field — agents are not slash commands.
**Consequences:** Establishes a third file type. Adds "Agent File Convention" section to AGENTS.md. Four agent files share consistent structure — sw-ship can load them uniformly.

---

### ADR: sw-ship branch and PR mechanics — auto-create vs. generate command

**Status:** Accepted
**Context:** "PR-ready branch with changelog" is the Ship phase output. The question is whether sw-ship executes `gh pr create` or generates the command for the user to run.
**Options:** (1) sw-ship runs `gh pr create` automatically — fast but potentially destructive (creates a PR without user review). (2) sw-ship generates the exact `gh pr create` command with title and body pre-filled — user confirms and runs it.
**Decision:** Generate the command, do not execute it. The "orchestrator suggests, never blocks" principle means the PR creation decision belongs to the user. sw-ship writes the PR title and body to the ship artifact, then outputs the ready-to-copy `gh pr create` command.
**Consequences:** User sees exactly what will be created before it is created. sw-ship does not need `Bash(gh *)` in allowed-tools — only `Read, Write, Glob, Grep, Bash(git diff *), Bash(git log *), Bash(git status)` for read-only git introspection.

---

## Implementation Sequence

Dependencies flow downward. Each phase can start when its dependencies are done.

### Phase 2A — Role Files (no dependencies)

**1. Create `skills/roles/developer.md`** — Developer role: pragmatic, Karpathy-disciplined voice; phases: [plan, build]; lead-phases: [build]
- Files: `skills/roles/developer.md` (create)
- Depends on: nothing
- Target: 80-120 lines

**2. Create `skills/roles/qa.md`** — QA role: adversarial voice + Release Prep section (Release Manager absorbed); phases: [build, ship]; lead-phases: [ship]
- Files: `skills/roles/qa.md` (create)
- Depends on: nothing
- Target: 80-120 lines
- Key constraint: Must have clearly delineated "Release Prep" subsection covering changelog entry, PR description, deployment notes, retrospective. This is an ADR decision from P1 — not optional.

### Phase 2B — Review Agent Definitions (no dependencies)

**3. Create `agents/review/correctness.md`** — Verifies logic correctness, edge cases, test coverage
- Files: `agents/review/correctness.md` (create)
- Depends on: nothing

**4. Create `agents/review/security.md`** — Checks OWASP-style vulnerabilities, input validation, auth surfaces
- Files: `agents/review/security.md` (create)
- Depends on: nothing

**5. Create `agents/review/simplicity.md`** — Checks for YAGNI violations, over-engineering, premature abstractions
- Files: `agents/review/simplicity.md` (create)
- Depends on: nothing

**6. Create `agents/review/performance.md`** — Checks for obvious performance issues (N+1, unnecessary allocations, blocking calls)
- Files: `agents/review/performance.md` (create)
- Depends on: nothing

### Phase 2C — sw-build Skill (depends on 2A)

**7. Create `skills/sw-build/SKILL.md`** — Developer-led build phase: reads plan artifact, implements tasks in TDD loop, QA companion after each task, writes build artifact
- Files: `skills/sw-build/SKILL.md` (create)
- Depends on: tasks 1, 2 (role files must exist so links are valid)
- Key sections: Role Loading (Developer lead, QA continuous), Context bootstrapping ($ARGUMENTS as plan path or text), Outcome (working code + build artifact), TDD loop activities, Build artifact template, Transition to sw-ship

### Phase 2D — sw-ship Skill (depends on 2B, 2C)

**8. Create `skills/sw-ship/SKILL.md`** — QA-led ship phase: reads build artifact + git diff, dispatches 4 parallel review agents, fix loop, Release Prep (changelog, PR command), writes ship artifact
- Files: `skills/sw-ship/SKILL.md` (create)
- Depends on: tasks 3-6 (agent files must exist so paths are valid), task 7 (sw-build must exist for transition)
- Key sections: Role Loading (QA lead, Developer support), Context bootstrapping, Review agent dispatch instructions (parallel), Fix loop, Release Prep, Ship artifact template

### Phase 2E — Updates to Existing Files (depends on 2C, 2D)

**9. Update `skills/sw-plan/SKILL.md`** — Replace "Build skill arrives in P2" placeholder with actual sw-build transition instruction
- Files: `skills/sw-plan/SKILL.md` (modify)
- Depends on: task 7 (sw-build must exist)
- Change: Replace the stub transition announcement with: "Plan ready at `docs/shipwright/plan/YYYY-MM-DD-<topic>.md`. Invoke `/sw-build docs/shipwright/plan/YYYY-MM-DD-<topic>.md` to begin implementation."

**10. Update `skills/sw-orchestrate/SKILL.md`** — Update routing table to reflect the full Discover → Plan → Build → Ship chain
- Files: `skills/sw-orchestrate/SKILL.md` (modify)
- Depends on: tasks 7, 8
- Change: Add Build and Ship to the phase reference table; clarify that users invoke sw-build and sw-ship explicitly after sw-plan/sw-discover complete.

**11. Update `AGENTS.md`** — Add `agents/review/` to directory structure; add Agent File Convention section; document `Bash(*)` exception pattern
- Files: `AGENTS.md` (modify)
- Depends on: tasks 3-6 (agent files should exist before documenting their convention)

**12. Bump version in `.claude-plugin/plugin.json`** — `0.1.0` → `0.2.0`
- Files: `.claude-plugin/plugin.json` (modify)
- Depends on: all other tasks (version bump is last)

---

## File Reference Guide

### New files (9)

| File | Type | Lines target |
|------|------|-------------|
| `skills/roles/developer.md` | role | 80-120 |
| `skills/roles/qa.md` | role | 80-120 |
| `agents/review/correctness.md` | agent | 40-60 |
| `agents/review/security.md` | agent | 40-60 |
| `agents/review/simplicity.md` | agent | 40-60 |
| `agents/review/performance.md` | agent | 40-60 |
| `skills/sw-build/SKILL.md` | skill | 150-220 |
| `skills/sw-ship/SKILL.md` | skill | 180-250 |

Missing count: only 8 files (not 9) — the plan file itself is the 9th item, already created.

### Modified files (4)

| File | Change |
|------|--------|
| `skills/sw-plan/SKILL.md` | Replace stub transition with sw-build invocation instruction |
| `skills/sw-orchestrate/SKILL.md` | Add Build/Ship to phase table |
| `AGENTS.md` | Add agents/ structure, agent file convention, Bash(*) exception note |
| `.claude-plugin/plugin.json` | Version bump to 0.2.0 |

---

## Content Specifications

### developer.md — Key content

```markdown
---
name: developer
type: role
voice: "Pragmatic, Karpathy-disciplined"
phases: [plan, build]
lead-phases: [build]
---

# Developer

## Voice
...

## Core Questions (before coding)
1. What is the simplest change that satisfies this task?
2. What assumption am I about to make that could be wrong?
3. What does the verification criterion say — am I building to that?
4. What existing code does this touch? (read before modifying)
5. How do I know when I'm done?

## Anti-Patterns
- Driving without a plan (implementing before understanding the task)
- Speculative abstractions (building for hypothetical future requirements)
- Non-surgical changes (touching code unrelated to the task)
- Implementing without a verification criterion
- Declaring done without running the test
```

**Quick-tier self-QA note:** When no QA companion is loaded, the Developer role includes a brief self-QA checklist (empty/null input, called twice, error path).

### qa.md — Key content

```markdown
---
name: qa
type: role
voice: "Adversarial, curious"
phases: [build, ship]
lead-phases: [ship]
---

# QA

## Voice
...

## Core Questions (adversarial testing)
1. What happens with empty, null, or enormous input?
2. What if this is called twice, or concurrently?
3. What does the user see when something fails?
4. Which assumption is most expensive if wrong?
5. What does the security surface look like from outside?

## Release Prep (Release Manager responsibilities)
When leading the Ship phase, QA shifts to a release preparation stance:

- **Changelog entry:** Summarize changes in user-facing language. Format: Keep a Changelog.
- **PR description:** Title (conventional commits format) + body (what, why, how to test, deployment notes).
- **Deployment notes:** Flag if any: env vars added/changed, migrations, dependency bumps, config changes.
- **Retrospective:** One paragraph: what went well, what was harder than expected, what to do differently. Save to docs/shipwright/ship/<artifact>.md.

## Anti-Patterns
- Scripted testing only (missing exploratory paths)
- Stopping at "it works on the happy path"
- Blocking progress over Low severity findings
- Skipping release prep because the code looks clean
```

### agent file template

```markdown
---
name: correctness
type: agent
perspective: correctness
severity-levels: [critical, high, medium, low]
---

# Correctness Review Agent

Review the provided code changes for logic errors, edge cases, and test coverage gaps.

## Focus

[Specific heuristics for this perspective]

## Heuristics

- [Heuristic 1]
- [Heuristic 2]
...

## Output Format

Return findings as:

**Correctness Review**

| Severity | Finding | File:Line | Recommendation |
|----------|---------|-----------|---------------|
| Critical | [description] | [file:line] | [fix] |
...

**Recommendation:** ship it / fix first
```

### sw-build SKILL.md structure

```
## Role Loading
[Developer](../roles/developer.md) (lead), [QA](../roles/qa.md) (continuous companion)

## Context
$ARGUMENTS = plan artifact path | free-text description | nothing (freestyle)

## Outcome
Required: working code with tests passing + build artifact at docs/shipwright/build/

## How to Get There
### 1. Plan Ingestion
### 2. TDD Loop (per task)
   a. Write test first
   b. Implement to pass
   c. Run verification criterion
   d. QA companion check
   e. Mark task complete in build artifact
### 3. Write Build Artifact

## Save Artifact
mkdir -p docs/shipwright/build/
Write docs/shipwright/build/YYYY-MM-DD-<topic>.md
Write BEFORE presenting transition

## Transition
"Build complete. Invoke `/sw-ship docs/shipwright/build/YYYY-MM-DD-<topic>.md` to review and ship."

## Build Artifact Template
---
type: build
topic: <slug>
tier: <quick|standard|major>
status: complete
date: YYYY-MM-DD
upstream: docs/shipwright/plan/YYYY-MM-DD-<topic>.md
---

## Tasks
| # | Description | Status | Notes |
|---|-------------|--------|-------|
| 1 | [task] | ✓ pass / ✗ fail | [issues noted] |

## Modified Files
- path/to/file.ext

## Issues Noted (QA companion)
- [severity]: [finding]
```

### sw-ship SKILL.md structure

```
## Role Loading
[QA](../roles/qa.md) (lead), [Developer](../roles/developer.md) (support in fix loop)

## Context
$ARGUMENTS = build artifact path | nothing (derive from git diff)

## Outcome
Required: all findings triaged + ship artifact + CHANGELOG.md appended + gh pr create command

## How to Get There
### 1. Context Bootstrapping
   Read build artifact (if provided) OR run git diff
### 2. Exploratory Testing (QA lead, adversarial)
### 3. Parallel Review Agent Dispatch
   Spawn 4 agents from agents/review/: correctness, security, simplicity, performance
   Input to each: git diff + plan artifact path
   Collect findings; triage by severity
### 4. Fix Loop (if blocking issues found)
   Developer role: fix → re-run relevant agent
   Never block: user can override
### 5. Release Prep (QA Release Manager stance)
   Write changelog entry
   Compose PR title + body
   Flag deployment notes if applicable
   Write retrospective
### 6. Write Ship Artifact

## Save Artifact
mkdir -p docs/shipwright/ship/
Write docs/shipwright/ship/YYYY-MM-DD-<topic>.md
Append to CHANGELOG.md at repo root (if present, using Keep a Changelog format)
Write BEFORE presenting transition

## Transition
"Ready to ship. Run:
gh pr create --title \"<title>\" --body-file docs/shipwright/ship/YYYY-MM-DD-<topic>.md"

## Ship Artifact Template
---
type: ship
topic: <slug>
tier: <quick|standard|major>
status: complete
date: YYYY-MM-DD
upstream: docs/shipwright/build/YYYY-MM-DD-<topic>.md
---

## Review Findings
[Consolidated agent output]

## PR Title
[Conventional commits format]

## PR Body
[What, why, how to test, deployment notes]

## Changelog Entry
[Keep a Changelog format]

## Retrospective
[What went well / what was harder / what to do differently]
```

---

## Acceptance Criteria

### Functional Requirements

- [ ] `skills/roles/developer.md` exists, 80-120 lines, follows role file convention (no `description` field, `type: role`)
- [ ] `skills/roles/qa.md` exists, 80-120 lines, contains clearly delineated "Release Prep" section
- [ ] `agents/review/correctness.md` exists with `type: agent` frontmatter and output template
- [ ] `agents/review/security.md` exists with `type: agent` frontmatter and output template
- [ ] `agents/review/simplicity.md` exists with `type: agent` frontmatter and output template
- [ ] `agents/review/performance.md` exists with `type: agent` frontmatter and output template
- [ ] `skills/sw-build/SKILL.md` exists, description is trigger-only (not workflow summary), loads Developer and QA roles via markdown links
- [ ] `skills/sw-ship/SKILL.md` exists, description is trigger-only, loads QA and Developer roles via markdown links, references all 4 agent files with correct paths
- [ ] `skills/sw-plan/SKILL.md` no longer contains the "Build skill arrives in P2" placeholder; contains actual sw-build transition instruction
- [ ] `skills/sw-orchestrate/SKILL.md` reflects full Discover → Plan → Build → Ship chain
- [ ] `AGENTS.md` directory structure section includes `agents/review/`; Agent File Convention section added
- [ ] `.claude-plugin/plugin.json` version is `0.2.0`

### Convention Compliance (AGENTS.md checklist)

- [ ] All new SKILL.md files: `name` matches directory name exactly
- [ ] All new SKILL.md files: `description` uses "Use when [triggers]" format — no workflow summary
- [ ] All new SKILL.md files: `allowed-tools` specified
- [ ] All new SKILL.md files: body uses imperative form, standard markdown headings, no XML tags
- [ ] All new SKILL.md files: under 500 lines
- [ ] All new role files: no `description` field in frontmatter
- [ ] All new role files: references to other files use markdown link syntax, not backticks
- [ ] All new agent files: `type: agent` in frontmatter, not `type: skill`
- [ ] sw-build and sw-ship: role loading uses markdown links (`[Developer](../roles/developer.md)`)
- [ ] sw-build and sw-ship: artifact written to disk BEFORE presenting transition options
- [ ] sw-build and sw-ship: transition passes explicit artifact path, not fuzzy slug

### Quality Gates

- [ ] `sw-plan → sw-build` transition instruction is coherent and uses the exact plan artifact path
- [ ] `sw-build → sw-ship` transition instruction is coherent and uses the exact build artifact path
- [ ] QA role `phases` frontmatter includes both `build` and `ship`
- [ ] QA role `lead-phases` includes only `ship`
- [ ] Developer role `phases` includes both `plan` and `build` (supports Architect in plan phase as a secondary voice)
- [ ] Review agent output format is consistent across all four agents (same severity levels, same table structure)
- [ ] sw-ship instructs passing the git diff as agent input — not the full conversation context

---

## Dependencies & Risks

### Dependencies

- P1 must be complete and committed on a branch (✓ done — `feat/shipwright-p1`, 2 commits ahead of `main`)
- P2 work should branch from `feat/shipwright-p1` or from `main` after P1 is merged

### Risk: Context budget with 4 parallel agents + 2 role files in sw-ship

sw-ship loads QA role + Developer role + dispatches 4 parallel agents. The 4 agents receive scoped input (diff + plan path) not the full session — this is the mitigation. Risk is low if agent input is bounded.

**Mitigation:** sw-ship SKILL.md must explicitly instruct agents to receive only the git diff (or targeted file contents) + plan artifact, not the full conversation context.

### Risk: Parallel agent spawning capability not verified in SKILL.md context

The P1 plan noted "skills cannot programmatically invoke other skills." Parallel review agents require Claude Code's Agent/Task tool from within a skill's execution. This is a distinct capability from skill-to-skill invocation.

**Mitigation:** If parallel dispatch is not possible from SKILL.md execution context, sw-ship falls back to sequential in-conversation review using QA role perspective shifts. The SKILL.md should note this fallback explicitly: "Dispatch parallel agents if supported; otherwise, review each perspective sequentially."

### Risk: sw-plan transition message change breaks existing users

Changing sw-plan's transition text is a backward-compatible change (better guidance, same functionality) — no functional regression. Low risk.

### Risk: QA role dual-voice coherence

QA plays adversarial tester in build phase and release manager in ship phase — cognitively different stances. The role file must model this as a phase shift (first adversarial, then release-prep) not simultaneous perspectives. If written poorly, Claude may blend the voices.

**Mitigation:** The role file should use a clear section break (`## Release Prep (Ship phase only)`) with an explicit stance-shift instruction: "In ship phase, after testing is complete, switch to the following release preparation stance."

---

## Future Considerations (P3)

- **Parallel build agents:** Independent tasks in the plan could be implemented by parallel sub-agents in separate worktrees. P2 defines the TDD loop sequentially; P3 adds the parallelism.
- **Learning loop automation:** sw-ship's retrospective is manually written in P2. P3 could auto-save retrospectives to `docs/shipwright/learn/` and index them for future discovery via a `sw-discover` extension.
- **Research agents:** `agents/research/` directory for web-search-backed discovery agents invokable from sw-discover.
- **Orchestrator intelligence:** sw-orchestrate currently uses static heuristics. P3 could add codebase-aware classification (read existing tests, migration files, etc. before assigning tier).

---

## Sources & References

### Origin

- **Origin document:** [docs/design.md](../../docs/design.md) — Section 3 (Phases), Section 4 (Roles), Section 5 (Project structure). Key decisions carried forward: QA absorbs Release Manager (ADR from P1 plan), parallel review agents (design spec section 3), TDD loop in build phase.

### Internal References

- P1 plan with governing ADRs: [docs/plans/2026-04-10-001-feat-shipwright-p1-plugin-foundation-plan.md](./2026-04-10-001-feat-shipwright-p1-plugin-foundation-plan.md)
- Skill conventions: [AGENTS.md](../../AGENTS.md)
- Cross-cutting rules: [CLAUDE.md](../../CLAUDE.md)
- Existing skill pattern: [skills/sw-plan/SKILL.md](../../skills/sw-plan/SKILL.md)
- Existing role pattern: [skills/roles/software-architect.md](../../skills/roles/software-architect.md) (contains canonical ADR template)
