---
name: sw-audit
description: >
  Use when the user has an existing messy, inherited, or chaotic codebase
  and needs to understand what they have before building anything new. Also
  use when the user says "I inherited this project", "the codebase is a mess",
  "I don't know where to start", "audit my project", "clean up this codebase",
  "someone else wrote this", or "help me understand what exists".
argument-hint: [project name or description, or nothing to audit current directory]
allowed-tools: Read, Write, Glob, Grep, Bash(git log *), Bash(ls *), Bash(find *), Bash(mkdir *)
---

# Shipwright Audit

Architect-led project scan: understand what exists, identify disorder, produce a prioritised triage roadmap, hand off to Plan → Build → Ship.

## Role Loading

Read and apply these role perspectives for this phase:

- [Software Architect](../roles/software-architect.md) (lead — scan and health assessment)
- [Product Manager](../roles/product-manager.md) (lead — triage and prioritisation)

If a role file cannot be loaded: "WARNING: Could not load [resource]. Proceeding with reduced capability."

## Context

If `$ARGUMENTS` is provided, treat it as a description of the project or the user's situation. Use it to focus the scan and interview.

If `$ARGUMENTS` is empty, audit the current working directory.

## Outcome

Produce an audit artifact containing ALL of these sections:

- **Project Map** — stack, structure, what each major module does
- **Health Signals** — test coverage, documentation, dependency freshness, dead code, commit cadence
- **Debt Hotspots** — specific files or areas that are high-complexity, inconsistent, or blocking progress
- **Triage Roadmap** — prioritised items, each formatted as a mini-scope ready for `/sw-plan`

## How to Get There

### 1. Scan

Apply the Software Architect perspective. Scan the project to build an objective picture of what exists.

**Determine scan depth first:**

```bash
find . -not -path '*/.git/*' -type f | wc -l
```

- **< 200 files:** deep scan — read key files in each module, not just top-level
- **≥ 200 files:** shallow scan — top-level structure, entry points, manifests only; list uninspected areas explicitly

**Always read:**

- Top-level directory structure (2 levels: `ls -la` and one level deeper)
- Package manifest (`package.json`, `Gemfile`, `pyproject.toml`, `go.mod`, `Cargo.toml` — whichever applies)
- `README.md` — age, completeness, accuracy
- Git log — commit cadence, contributor count, last activity: `git log --oneline --since="6 months ago" | wc -l` and `git log --oneline -10`
- Test directory — presence, framework, rough coverage signal
- Entry points — `main.*`, `app.*`, `index.*`, `server.*`, `routes.*`

**Flag during scan (do not ask the user yet):**

- Duplicate patterns (two auth systems, two routing approaches, etc.)
- Large files (>300 lines in a scripting language) or deeply nested directories
- Commented-out code blocks or TODO/FIXME density
- Dependencies with no obvious use
- Folders with no recent commits (>6 months)
- Missing infrastructure: no tests, no CI config, no error handling patterns

### 2. Targeted Interview

After the scan, identify the top ambiguities — things the code cannot answer. Ask about them **one at a time**, in order of importance. Stop after 5 questions maximum, or earlier if ambiguities are resolved.

Ask only when the answer materially changes the triage. Skip questions where the answer doesn't affect the roadmap.

Example triggers and questions:

| What the scan found | Question to ask |
|---------------------|-----------------|
| Two parallel implementations of the same concept | "I see both `X` and `Y` handling [concern] — which is the intended direction going forward?" |
| Large folder with no recent commits | "The `legacy/` folder hasn't changed in [N] months — is this intentionally frozen, or abandoned mid-way?" |
| Near-zero test coverage | "Test coverage appears very low — is adding tests a goal for this project, or is it intentionally untested?" |
| Half-implemented feature | "There's a partial implementation of [feature] in `path/` — is this in progress or abandoned?" |
| No README or severely outdated README | "The README is [missing / from YYYY] — is there documentation elsewhere, or should we treat the code as the source of truth?" |

If the user answers "I don't know" — mark that area `status: unknown` in the audit artifact. Never block on an unanswered question.

### 3. Triage

Apply the Product Manager perspective. Synthesise scan findings and interview answers into a prioritised triage roadmap.

**Categories:**

| Category | What it covers |
|----------|---------------|
| Cleanup | Remove dead code, consolidate duplicates, rename for clarity |
| Stabilisation | Add tests, add error handling, document existing behaviour |
| Infrastructure | CI, linting, dependency updates, missing tooling |
| New Feature | Capabilities the project needs but doesn't have yet |

**Prioritisation heuristics:**

- Rank Cleanup and Stabilisation above New Feature — you can't reliably build on unstable ground
- Rank Infrastructure above everything if there is no CI or no tests at all
- Within a category, rank by: blocking other work > high user impact > low effort

**Each roadmap item must include:**

- Title and category
- One-sentence rationale (why this, why now)
- Effort signal: Low / Medium / High
- `/sw-plan` invocation path (so the user can act immediately)

### 4. Save Artifact

Before presenting the roadmap:

```bash
mkdir -p docs/shipwright/audit/
```

Save the audit document to `docs/shipwright/audit/YYYY-MM-DD-<project-slug>.md`.

Derive `<project-slug>` from the project's directory name or the user's description. Replace today's date for `YYYY-MM-DD`.

Use this frontmatter:

```yaml
---
type: audit
topic: <project-slug>
tier: major
status: complete
date: YYYY-MM-DD
upstream: null
---
```

Write the artifact to disk before presenting the roadmap.

### 5. Transition

After saving, present the triage roadmap. Highlight the suggested first item (highest priority). Announce:

> Audit complete. Roadmap saved to `docs/shipwright/audit/YYYY-MM-DD-<project-slug>.md`.
>
> Suggested starting point: **[item title]** — [one-sentence rationale].
>
> Run `/sw-plan docs/shipwright/audit/YYYY-MM-DD-<project-slug>.md` to begin, or pick any item from the roadmap.

The user can also:

- Pick a different roadmap item to start with
- Stop here and use the audit artifact as a reference
- Skip to build directly without planning

## Audit Artifact Template

```markdown
---
type: audit
topic: <project-slug>
tier: major
status: complete
date: YYYY-MM-DD
upstream: null
---

# Audit: <Project Name>

## Project Map

**Stack:** [language, framework, key dependencies]

**Structure:**
[Top-level directory summary — what each major directory contains]

**Entry points:** [main files where execution or request handling begins]

## Health Signals

| Signal | Status | Detail |
|--------|--------|--------|
| Tests | ✓ present / ✗ absent / ⚠ partial | [framework, rough coverage] |
| Documentation | ✓ current / ⚠ outdated / ✗ absent | [README age, inline doc quality] |
| Dependencies | ✓ current / ⚠ stale / ✗ unknown | [manifest present, last updated] |
| Commit cadence | ✓ active / ⚠ slow / ✗ stalled | [commits in last 6 months] |
| Dead code | ✓ none / ⚠ some / ✗ significant | [specific areas if known] |
| CI / tooling | ✓ present / ✗ absent | [CI config, linter, formatter] |

## Debt Hotspots

- **[File or area]** — [why it's a hotspot: complexity, inconsistency, duplication, etc.]
- **[File or area]** — [description]

*Areas not inspected (shallow scan):* [list if applicable, or "None — full scan completed"]

## Triage Roadmap

### Priority 1 — [Category]

**[Item title]**
- Rationale: [one sentence]
- Effort: Low / Medium / High
- Start: `/sw-plan docs/shipwright/audit/YYYY-MM-DD-<project-slug>.md`

### Priority 2 — [Category]

**[Item title]**
- Rationale: [one sentence]
- Effort: Low / Medium / High
- Start: `/sw-plan docs/shipwright/audit/YYYY-MM-DD-<project-slug>.md`

[Continue for all roadmap items]
```

## Principles

- Scan first, ask second. Never ask a question the code can already answer.
- Ask one question at a time. Five questions maximum — prioritise ruthlessly.
- Mark unknowns explicitly. "Status: unknown" is a valid audit finding.
- Suggest, never block. If the user wants to skip the interview or the triage, let them.
- Stability before features. The roadmap should default to "fix the foundation before adding floors."
- Name uninspected areas. A shallow scan that pretends to be complete is worse than one that flags its own gaps.
