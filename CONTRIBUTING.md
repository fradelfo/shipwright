# Contributing to Shipwright

Shipwright is a Claude Code plugin — a collection of skill files, agent files, and role files that Claude reads and follows. There's no build step, no test runner, no binary. Contributing means writing markdown that Claude executes.

This guide explains how the pieces fit together and what to check before opening a PR.

---

## How It Works

### Skills

Skills are slash commands. When a user runs `/sw-discover`, Claude Code loads `skills/sw-discover/SKILL.md` and follows it as an instruction set.

Each skill file has YAML frontmatter and a markdown body:

```yaml
---
name: sw-discover           # Must match the directory name exactly
description: >              # Trigger description — "Use when [situation]"
  Use when the user needs...
argument-hint: [hint]       # Shown in autocomplete
allowed-tools: Read, Write  # Principle of least privilege
---
```

The body uses imperative/infinitive form ("Read the file", not "You should read the file") and standard markdown headings.

### Agents

Agents are sub-agent instruction sets dispatched by skills. They are NOT slash commands — users never invoke them directly. They live in `agents/<category>/`.

```yaml
---
name: codebase-analyst
type: agent
perspective: codebase       # What lens this agent applies
severity-levels: [critical, high, medium, low]  # If applicable
---
```

Agents have no `description` field (they're not invokable). They have a one-paragraph focus statement, a heuristics checklist, and an output template. Target 40–60 lines.

### Roles

Roles are passive persona definitions. Skills load them via markdown links:

```markdown
- [Software Architect](../roles/software-architect.md) (lead role)
```

Claude reads the role file and applies that perspective for the duration of the phase. Roles are never invoked directly.

```yaml
---
name: software-architect
type: role
voice: "Systematic, trade-off-aware"
phases: [discover, plan]
lead-phases: [plan]
---
```

---

## Adding a New Skill

1. Create `skills/sw-<name>/SKILL.md`
2. Run the compliance checklist below
3. Update `AGENTS.md` — add the new directory to the structure
4. Update `README.md` — add to Phases table and Usage section if user-facing
5. Bump `plugin.json` version (patch for fixes, minor for new skills)
6. Add a routing trigger to `sw-orchestrate/SKILL.md` if the skill has natural entry conditions

### Skill Compliance Checklist

Before opening a PR for a new or modified skill:

- [ ] `name` in frontmatter matches directory name exactly (`sw-foo` → `skills/sw-foo/SKILL.md`)
- [ ] `description` uses "Use when [situation]" format — describes triggers, not the workflow
- [ ] `description` includes natural-language phrases users would say
- [ ] `allowed-tools` lists only tools the skill actually needs (principle of least privilege)
- [ ] Body uses imperative form ("Read the file"), not second person ("You should read")
- [ ] Body uses standard markdown headings, not XML tags
- [ ] File is under 500 lines
- [ ] Role references use markdown link syntax: `[Name](../roles/name.md)`, not backticks
- [ ] Artifacts write to `docs/shipwright/<phase>/YYYY-MM-DD-<topic>.md` with YAML frontmatter
- [ ] Transition section passes explicit artifact path (no fuzzy slug matching)
- [ ] "Suggests, never blocks" — every mandatory step has an override or fallback path

**Exception — `Bash(*)`:** `sw-build` uses `Bash(*)` rather than specific patterns. This is an intentional, documented exception: test commands vary per project stack. All other skills must enumerate specific Bash patterns.

---

## Adding a New Agent

1. Create `agents/<category>/<name>.md`
2. Frontmatter: `name`, `type: agent`, `perspective` — no `description` field
3. Body: one-paragraph focus statement, heuristics checklist, output template
4. Target 40–60 lines
5. Update `AGENTS.md` directory structure
6. Update the dispatching skill to reference the new agent via markdown link

Agents receive scoped input from the calling skill (a diff, a code unit, a plan path). Write the agent assuming it won't have full conversation context.

---

## Adding or Modifying a Role

Roles are loaded by skills via markdown links. Changing a role propagates to every skill that uses it — be careful with voice changes.

- No `description` field in frontmatter
- Target 80–120 lines — Claude already knows the domain; specify only what's unique to this role
- Include: voice, core questions, output format, phase behaviour, anti-patterns

If you're adding a new role, also update any skill that should load it.

---

## The Core Principles (Apply to All Contributions)

**Suggests, never blocks.** Every mandatory step must have an override path. Users who want to skip the interview, jump to build, or ignore a QA finding can always do so. Record the override in the artifact, but never hard-block.

**Simplicity first.** No speculative abstractions. A new agent file for a one-time use case is wrong — extend an existing agent or handle it inline. Three clear lines of instruction beat a flexible but complex framework.

**Artifacts live in the user's repo.** Shipwright is stateless. It produces markdown documents in `docs/shipwright/`. Nothing is stored in the plugin itself.

**Principle of least privilege.** `allowed-tools` should be the minimum needed. If a skill only needs to read files, it shouldn't have `Bash(*)`.

**Error message convention.** When a resource cannot be loaded: `"WARNING: Could not load [resource]. Proceeding with reduced capability."` — consistent, non-blocking.

---

## Testing Your Changes

There's no automated test suite — the skill files are markdown, not code. Testing means:

1. **Run the compliance checklist** on any skill you added or modified
2. **Read the skill as Claude would** — follow it step by step mentally and verify the output makes sense
3. **Check cross-references** — if a skill references an agent or role via markdown link, verify the path resolves correctly from the skill's location
4. **Run it live** — load the plugin in Claude Code and invoke the skill on a sample project. The fastest way to find issues is to use it.

---

## Artifact Frontmatter Reference

All artifacts written to `docs/shipwright/` must include:

```yaml
---
type: audit | discover | plan | build | ship | learn
topic: <kebab-case-slug>
tier: quick | standard | major
status: complete
date: YYYY-MM-DD
upstream: <path-to-upstream-artifact, or null>
---
```

The `upstream` field creates a traceable chain: ship → build → plan → discover/audit.

---

## Commit Messages

Use conventional commits:

```
feat(sw-audit): add rescue tier for existing codebases
fix(sw-ship): correct artifact path in transition announcement
docs(methodology): add learning loop explanation
```

Scope is the skill or component name (`sw-audit`, `sw-orchestrate`, `roles`, `agents`, `docs`).

---

## Questions

Open an issue on GitHub. If you're unsure whether a change fits the methodology, describe what you're trying to add and why — the design principles in [docs/methodology.md](docs/methodology.md) are the reference point.
