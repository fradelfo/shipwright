---
title: "feat: Shipwright P3 — Compounding Intelligence"
type: feat
status: completed
date: 2026-04-10
origin: docs/design.md
---

# feat: Shipwright P3 — Compounding Intelligence

## Overview

P1 added product and design thinking. P2 completed the build/ship lifecycle. P3 makes the system smarter over time and faster on large tasks: research agents that surface context before planning, QA agents that test systematically, a learning loop that captures retrospectives and makes them discoverable, and parallel worktree orchestration for independent build tasks.

After P3, Shipwright compounds: each ship makes the next discovery better, and large plans can be built faster by running independent tasks simultaneously.

Origin: [docs/design.md](../../docs/design.md) — Section 7 (Phased Release Plan), Section 3 (Phase 3 Build, Phase 4 Ship), Section 5 (agents/qa/, agents/research/).

---

## Architecture Decisions

### ADR: Research agents — inline personas vs. dispatched sub-agents

**Status:** Accepted
**Context:** The four review agents in sw-ship are applied as inline instruction sets (the agent file is read and the perspective is applied in-conversation). Research agents (codebase-analyst, docs-researcher) need to read files and potentially fetch URLs — operations that require tool access. Two models are possible.
**Options:** (1) Sub-agent dispatch: each agent file is a full sub-agent with its own tool permissions. Requires `allowed-tools` in agent frontmatter — a format divergence from the existing four agents. (2) Inline persona: sw-plan reads the agent file as an instruction set and applies it in-conversation using sw-plan's own tool permissions.
**Decision:** Inline persona (Option 2). Consistent with existing agent format — no `allowed-tools` field on agent files. The calling skill (sw-plan) adds `WebFetch` and `Grep` to its own `allowed-tools` to support docs-researcher and codebase-analyst respectively.
**Consequences:** Agent files remain format-consistent across all categories. sw-plan's `allowed-tools` expands. No format divergence to document or maintain.

---

### ADR: Learn file searchability — index vs. grep vs. glob-and-read

**Status:** Accepted
**Context:** sw-discover needs to surface relevant past learnings before problem framing. Three mechanisms were considered.
**Options:** (1) Full-text grep on learn/ directory — requires adding Bash(grep) to sw-discover. (2) Frontmatter tag matching — requires sw-ship to populate tags intelligently; requires reading frontmatter only. (3) Glob all learn/ files, read them, Claude assesses relevance inline — no extra tooling, works for small collections.
**Decision:** Option 3 — Glob and read, inline relevance assessment. The learn/ directory is expected to be small (tens of files, not thousands). This is the simplest design that handles the requirement. If the collection grows large enough to make full reads expensive, a future iteration can introduce an index.
**Consequences:** sw-discover's `allowed-tools` gains `Glob`. No index file to maintain. Scales to ~100 files without performance issues.

---

### ADR: Parallel worktree trigger — automatic vs. explicit

**Status:** Accepted
**Context:** The design spec says "for independent modules identified in the plan, spawn parallel agents. Each gets its own worktree." The question is whether sw-build detects parallelisability automatically or requires explicit opt-in.
**Options:** (1) Automatic detection — sw-build analyses task dependencies at plan ingestion and decides. Risky: a wrong automatic split could create merge conflicts or lost work. (2) Explicit opt-in — the plan artifact must include a `parallel: true` flag on task groups, or the user must request it.
**Decision:** Explicit opt-in. The plan must annotate task groups as parallelisable (either by the Architect in sw-plan or by the user at build time). sw-build does not auto-detect. The "suggests, never blocks" principle applies: sw-build may suggest parallelism when it detects likely independence, but does not act without confirmation.
**Consequences:** No false parallel splits. Plan artifacts get a new optional annotation syntax. Slightly more friction to use the feature, but far safer. The Architect role in sw-plan should be updated to flag parallelisable tasks.

---

### ADR: edge-case-hunter — supplements or replaces QA Companion questions

**Status:** Accepted
**Context:** The QA Companion in sw-build currently asks five ad-hoc questions after each task (empty/null input, idempotency, concurrency, error path, most expensive assumption). edge-case-hunter would do similar work more systematically.
**Options:** (1) Replace — edge-case-hunter subsumes the five questions entirely. Simpler, but removes fallback behavior when sub-agent dispatch is not supported. (2) Supplement — five questions run in-conversation first (fast, always available), then edge-case-hunter is dispatched for deeper analysis.
**Decision:** Supplement (Option 2). The five in-conversation questions are instant and always available — they don't require sub-agent dispatch. edge-case-hunter provides systematic depth on top. Both Critical/High findings from either source block the next task. Fallback: if dispatch is not supported, the five questions serve as the complete QA companion.
**Consequences:** QA Companion step in sw-build gains one sub-step. Slightly more steps per task, but depth increase justifies it.

---

## Implementation Sequence

Dependencies: Phase 3A (new agent files) must complete before Phase 3D and 3E (which reference them). Phase 3B (sw-ship learn write) should complete before Phase 3C (sw-discover learn read — need entries to discover). Phases 3D and 3E can run in parallel with each other.

### Phase 3A — New Agent Files (no dependencies)

**1. Create `agents/research/codebase-analyst.md`** — Reviews the existing codebase and extracts patterns, conventions, and relevant files relevant to the planned feature.
- Files: `agents/research/codebase-analyst.md` (create)
- Depends on: nothing
- Format: same frontmatter schema as review agents (`name`, `type: agent`, `perspective: codebase`). Body: Input (feature description + codebase root), Approach (what to survey), Output Format (patterns inventory).
- Note: no `severity-levels` — this agent produces findings, not severity judgements.

**2. Create `agents/research/docs-researcher.md`** — Fetches and summarises external documentation for libraries, frameworks, or APIs referenced in the plan.
- Files: `agents/research/docs-researcher.md` (create)
- Depends on: nothing
- Fallback: if internet unavailable, scan local docs (README, package manifests, type definitions) and emit WARNING.

**3. Create `agents/qa/edge-case-hunter.md`** — Systematic edge-case generation for a single code unit after task completion.
- Files: `agents/qa/edge-case-hunter.md` (create)
- Depends on: nothing
- Input: the just-written code + task's verification criterion. Not a diff — a targeted unit.
- Uses `severity-levels` — findings can block next task.

**4. Create `agents/qa/exploratory-tester.md`** — End-to-end "confused user" exploration of the completed change at ship time.
- Files: `agents/qa/exploratory-tester.md` (create)
- Depends on: nothing
- Input: the full git diff + build artifact (for intent context).
- Output: narrative of what was tried + severity-labelled findings table.

### Phase 3B — sw-ship: learn write path + exploratory-tester dispatch

**5. Update `skills/sw-ship/SKILL.md`** — Two additions:
- (a) In Step 2 (Exploratory Testing): dispatch `[Exploratory Tester](../../agents/qa/exploratory-tester.md)` explicitly (currently ad-hoc inline QA)
- (b) After Save Artifact: add a Learn Write step — `mkdir -p docs/shipwright/learn/`, write `docs/shipwright/learn/YYYY-MM-DD-<topic>.md` using the learn file template
- Files: `skills/sw-ship/SKILL.md` (modify)
- Depends on: tasks 1-4 (agent files must exist for links to be valid)

### Phase 3C — sw-discover: past learnings read path

**6. Update `skills/sw-discover/SKILL.md`** — Add a Past Learnings step before Problem Framing:
- Glob `docs/shipwright/learn/` for all `.md` files
- If directory empty or missing: silently skip — begin Problem Framing immediately
- If entries exist: read the most recent 5, assess relevance to current topic inline, surface relevant ones to the user as "Past learnings from similar work" before the PM role begins
- Add `Glob` to `allowed-tools`
- Files: `skills/sw-discover/SKILL.md` (modify)
- Depends on: task 5 (learn files must be writable before discover can surface them — logical dependency for testing)

### Phase 3D — sw-plan: research agent dispatch

**7. Update `skills/sw-plan/SKILL.md`** — Add Research Agent Dispatch section as the first activity in "How to Get There", before Codebase Analysis:
- Dispatch `[Codebase Analyst](../../agents/research/codebase-analyst.md)` to survey existing patterns
- Dispatch `[Docs Researcher](../../agents/research/docs-researcher.md)` if the plan references external libraries/APIs (conditional)
- Inject agent findings as context before the Architect role produces its output
- Add `WebFetch, Grep` to `allowed-tools`
- Files: `skills/sw-plan/SKILL.md` (modify)
- Depends on: tasks 1, 2 (agent files must exist)

### Phase 3E — sw-build: parallel orchestration + edge-case-hunter dispatch

**8. Update `skills/sw-build/SKILL.md`** — Two additions:
- (a) In Step 1 (Plan Ingestion): add Parallel Task Analysis — identify task groups flagged as parallel in the plan artifact; confirm with user before splitting; dispatch sub-agents into worktrees if confirmed; orchestrator owns single build artifact; fallback to sequential if dispatch not supported
- (b) In Step 3 (QA Companion Activation): after the five in-conversation questions, dispatch `[Edge Case Hunter](../../agents/qa/edge-case-hunter.md)` with the just-written code and task verification criterion
- Files: `skills/sw-build/SKILL.md` (modify)
- Depends on: task 3 (edge-case-hunter must exist)

### Phase 3F — Housekeeping

**9. Update `AGENTS.md`** — Add `agents/qa/` and `agents/research/` to the directory structure diagram.
- Files: `AGENTS.md` (modify)
- Depends on: tasks 1-4 (agent files should exist before being documented)

**10. Bump `plugin.json` version** — `0.2.0` → `0.3.0`
- Files: `.claude-plugin/plugin.json` (modify)
- Depends on: all other tasks

---

## Content Specifications

### Learn File Template (`docs/shipwright/learn/YYYY-MM-DD-<topic>.md`)

```markdown
---
type: learn
topic: <topic-slug>
date: YYYY-MM-DD
upstream: docs/shipwright/ship/YYYY-MM-DD-<topic>.md
tags: [tag1, tag2, tag3]
---

# Learning: <Topic>

## Retrospective

[Retrospective paragraph from ship artifact — copied verbatim]

## Key Insights

- [Distilled, actionable insight — one sentence]
- [Second insight if applicable]
```

**Tags:** Derived from the topic slug (split on hyphens) plus any domain words from the retrospective (e.g., "auth", "api", "performance"). QA role in Release Prep stance writes them.

**sw-discover surfacing format** (when past learnings are relevant):

```markdown
> **Past learnings from similar work:**
> - [YYYY-MM-DD] [Topic]: [Key insight one sentence]
> - [YYYY-MM-DD] [Topic]: [Key insight one sentence]
>
> Proceeding to Problem Framing with this context in mind.
```

---

### codebase-analyst.md — Key content

```markdown
---
name: codebase-analyst
type: agent
perspective: codebase
---

# Codebase Analyst Agent

Survey the existing codebase for patterns, conventions, and relevant files
that apply to the planned feature. Produce a concise pattern inventory for
the Architect to reference before making design decisions.

## Input
- Feature description or discovery doc summary
- Codebase root path

## Approach
- Directory structure (top 2 levels)
- Naming conventions (file names, function names, module names)
- Technology stack (framework, language version, key dependencies)
- Relevant existing files (pattern matches to the planned feature)
- Coding style signals (test framework, linting conventions)
- Any existing abstractions the plan should extend or avoid duplicating

## Output Format
**Codebase Pattern Inventory**
- Stack: [language, framework, key deps]
- Conventions: [naming, file organisation]
- Relevant files: [path:line — what to read before implementing]
- Extend: [existing abstractions to reuse]
- Avoid: [existing patterns the plan should not duplicate]
```

---

### docs-researcher.md — Key content

```markdown
---
name: docs-researcher
type: agent
perspective: documentation
---

# Docs Researcher Agent

Fetch and summarise current documentation for external libraries, frameworks,
or APIs referenced in the plan. Prioritise recent versions.
Fallback to local docs if internet is unavailable.

## Input
- Library or API name(s) from the plan
- Specific questions or patterns to look up

## Approach
1. Scan local docs first: README, package manifests, type definitions
2. Attempt external fetch for official documentation
3. If fetch fails: emit WARNING and report local findings only

## Fallback
WARNING: Could not reach external docs for [library].
Local documentation found: [summary]. Proceeding with reduced capability.

## Output Format
**Documentation Summary: [Library]**
- Version: [relevant version]
- Key patterns: [how to use the relevant API]
- Gotchas: [version-specific or common mistakes]
- References: [URLs fetched, or "local only"]
```

---

### edge-case-hunter.md — Key content

```markdown
---
name: edge-case-hunter
type: agent
perspective: edge-cases
severity-levels: [critical, high, medium, low]
---

# Edge Case Hunter Agent

Systematically generate edge cases for a single completed code unit.
Receives the just-written code and the task's verification criterion.
Produces findings that block (Critical/High) or advise (Medium/Low).

## Input
- The just-written code (not a diff — the targeted unit)
- The task's verification criterion from the plan

## Heuristics
[Systematic edge-case categories...]

## Output Format
| Severity | Edge Case | Reproduction | Recommendation |
...
Recommendation: proceed / fix before next task
```

---

### Parallel Worktree Protocol (for sw-build SKILL.md)

```markdown
### Parallel Execution (opt-in)

If the plan artifact contains task groups annotated with `[parallel]`, or the
user requests parallel execution:

1. **Confirm with user** — "Tasks [N, M] appear independent. Run them in parallel
   worktrees? This is faster but requires git worktree support."
2. **Create worktrees** — One per parallel group:
   git worktree add ../sw-<topic>-task<N> HEAD
3. **Dispatch sub-agents** — One per worktree, each receives:
   - The plan artifact path (for overall context)
   - The specific task group (Goal, Files, Verification criterion)
   - The worktree path as working directory
4. **Collect results** — Sub-agents return task completion status and QA
   findings via Task tool output. The orchestrating agent writes the
   unified build artifact.
5. **Clean up worktrees** — After all groups complete:
   git worktree remove ../sw-<topic>-task<N>

**Fallback:** If Task dispatch is not supported, execute task groups sequentially.
The "parallel" annotation is advisory — never blocks.
**Conflict protocol:** If a sub-agent fails, it reports the failure. The
orchestrator surfaces it to the user before proceeding.
```

---

## File Reference Guide

### New files (4)

| File | Type | Lines target |
|------|------|-------------|
| `agents/research/codebase-analyst.md` | agent | 45-55 |
| `agents/research/docs-researcher.md` | agent | 45-55 |
| `agents/qa/edge-case-hunter.md` | agent | 45-55 |
| `agents/qa/exploratory-tester.md` | agent | 45-55 |

### Modified files (6)

| File | Change |
|------|--------|
| `skills/sw-plan/SKILL.md` | Add Research Agent Dispatch (step 0 of How to Get There); add WebFetch, Grep to allowed-tools |
| `skills/sw-build/SKILL.md` | Add Parallel Execution section in Step 1; extend QA Companion to dispatch edge-case-hunter |
| `skills/sw-ship/SKILL.md` | Extend Step 2 to dispatch exploratory-tester; add Learn Write step after Save Artifact |
| `skills/sw-discover/SKILL.md` | Add Past Learnings step before Problem Framing; add Glob to allowed-tools |
| `AGENTS.md` | Add agents/qa/ and agents/research/ to directory structure |
| `.claude-plugin/plugin.json` | Version 0.2.0 → 0.3.0 |

---

## Acceptance Criteria

### Functional Requirements

- [ ] `agents/research/codebase-analyst.md` exists, `type: agent`, `perspective: codebase`, no `description` field, no `severity-levels` (it produces findings, not severity judgements)
- [ ] `agents/research/docs-researcher.md` exists, `type: agent`, `perspective: documentation`, includes fallback behavior for internet failure
- [ ] `agents/qa/edge-case-hunter.md` exists, `type: agent`, `perspective: edge-cases`, includes `severity-levels`, input is targeted code unit + verification criterion (not a diff)
- [ ] `agents/qa/exploratory-tester.md` exists, `type: agent`, `perspective: exploratory`, includes `severity-levels`, input is the full git diff + build artifact
- [ ] `skills/sw-plan/SKILL.md` references codebase-analyst and docs-researcher via markdown links with correct relative paths
- [ ] `skills/sw-plan/SKILL.md` `allowed-tools` includes `WebFetch` and `Grep`
- [ ] `skills/sw-build/SKILL.md` references edge-case-hunter via markdown link with correct relative path
- [ ] `skills/sw-build/SKILL.md` includes Parallel Execution section with worktree protocol and sequential fallback
- [ ] `skills/sw-ship/SKILL.md` references exploratory-tester via markdown link with correct relative path
- [ ] `skills/sw-ship/SKILL.md` includes a Learn Write step after Save Artifact (`mkdir -p docs/shipwright/learn/`, writes learn file)
- [ ] `skills/sw-discover/SKILL.md` has a Past Learnings step before Problem Framing — silently skips if learn/ empty
- [ ] `skills/sw-discover/SKILL.md` `allowed-tools` includes `Glob`
- [ ] `AGENTS.md` directory structure shows `agents/qa/` and `agents/research/`
- [ ] `.claude-plugin/plugin.json` version is `0.3.0`

### Convention Compliance

- [ ] All new agent files: `type: agent`, no `description` field (not slash commands)
- [ ] All new agent files: 45-55 lines
- [ ] All new agent files: Input, Approach/Heuristics, Output Format sections present
- [ ] All skill edits: added sections use imperative form, standard markdown headings
- [ ] All new agent file references from skills: relative markdown links, correct paths
  - From sw-plan: `../../agents/research/codebase-analyst.md`
  - From sw-build: `../../agents/qa/edge-case-hunter.md`
  - From sw-ship: `../../agents/qa/exploratory-tester.md`
- [ ] sw-ship learn write step: `mkdir -p` before writing, write before announcing transition
- [ ] sw-discover learn read step: silently skips (no error message) when learn/ is empty or missing

### Artifact Integrity

- [ ] Learn file template has frontmatter: `type: learn`, `topic`, `date`, `upstream`, `tags`
- [ ] Learn file body: Retrospective paragraph + Key Insights bullets
- [ ] Learn file `upstream` field points to the ship artifact (not the plan artifact)
- [ ] sw-ship ship artifact `upstream` field continues to point to build artifact (unchanged)
- [ ] Parallel worktree protocol: orchestrator owns single build artifact; sub-agents do not write their own

---

## Risk Flags

- **docs-researcher internet access** — Claude Code may not have WebFetch access in all execution environments. Mitigation: local-docs-first approach + explicit WARNING fallback is already specified.
- **Parallel worktree file conflicts** — if two parallel task groups touch the same file, the merge will conflict. Mitigation: the Architect role in sw-plan should not flag tasks as `[parallel]` if their file scopes overlap. sw-build warns the user if it detects overlap before creating worktrees.
- **learn/ growing large over time** — Glob-and-read-all approach degrades at scale. Mitigation: the plan explicitly defers this to a future iteration. At 100 files (~2 years of weekly ships), performance is still acceptable; at 1,000 files, an index would be warranted.
- **sw-discover allowed-tools expansion** — Adding `Glob` to sw-discover changes its tool permissions. Verify this does not create unintended access. Mitigation: `Glob` is read-only and path-based — no write capability implied.
- **Skill file size creep** — sw-ship already has 249 lines. Adding exploratory-tester dispatch and the learn write step will push it toward the 500-line limit. Mitigation: keep the additions concise; the parallel review agent dispatch pattern established in P2 shows that 10-15 lines suffice for each addition.

---

## Future Considerations (beyond P3)

- **Learn index file** — if learn/ grows beyond ~100 entries, a structured `index.md` with frontmatter summaries per entry would make sw-discover faster.
- **Cross-project learning** — currently learnings are per-project. A future shared learn/ registry (e.g., `~/.shipwright/learn/`) would allow learnings to carry across codebases.
- **Automated tagging** — currently tags in learn files are written by the QA Release Prep stance. A dedicated tag-suggestion step could improve tag consistency.
- **Parallel worktree UX** — the current opt-in model requires plan annotation. A future iteration could add an sw-orchestrate signal ("this plan has N independent modules — parallelize?") before build begins.

---

## Sources & References

### Origin

- **Origin document:** [docs/design.md](../../docs/design.md) — Section 7 (P3 scope), Section 3 Phase 3 and 4 (build parallelism, ship retrospective). Key decisions carried forward: research agents before planning, QA agents in build+ship, learning loop at ship time surfaced at discover time, parallel worktrees for independent build tasks.

### Internal References

- P2 plan (governing ADRs): [docs/plans/2026-04-10-002-feat-shipwright-p2-build-ship-phases-plan.md](./2026-04-10-002-feat-shipwright-p2-build-ship-phases-plan.md)
- Agent file format: [agents/review/correctness.md](../../agents/review/correctness.md)
- Parallel review agent dispatch pattern: [skills/sw-ship/SKILL.md](../../skills/sw-ship/SKILL.md) (Step 3)
- Skill conventions: [AGENTS.md](../../AGENTS.md)
