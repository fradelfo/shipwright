# Shipwright — Design Specification

> "Build the whole ship."

**Status:** v0.3.0 — complete
**Author:** Francesco
**Date:** 2026-04-10

---

## 1. Identity & Positioning

**Shipwright** is an opinionated development methodology for Claude Code that optimises for shipping products, not just writing code.

**Core thesis:** Existing Claude Code skills optimise for coding. Shipwright optimises for shipping products. Products need product thinking, design thinking, and quality thinking — not just code. The difference between a feature and a shipped feature is the work that happens before and after implementation.

**Target audience:** Developers using Claude Code who work on non-trivial projects — side projects, startups, open source, professional work. Anyone who's felt "Claude wrote the code fine but built the wrong thing" or "it coded fast but the result was fragile."

**What it is NOT:**
- Not a replacement for project management tools (Linear, Jira)
- Not a CI/CD system
- Not framework-specific (works with any stack)

**Inspired by:**
- [superpowers](https://github.com/anthropics/claude-code-plugins) — workflow discipline, brainstorming-before-code, TDD, verification
- [compound-engineering](https://github.com/nichochar/compound-engineering) — knowledge compounding, multi-agent orchestration, plan-work-review lifecycle
- [Karpathy Guidelines](https://github.com/forrestchang/andrej-karpathy-skills) — simplicity-first, surgical changes, goal-driven execution

---

## 2. The Orchestrator — Tiered Complexity

The heart of Shipwright. Every user request first hits the orchestrator, which classifies task complexity and routes to the appropriate lifecycle depth.

### Three tiers

| Tier | Signal | Lifecycle | Example |
|---|---|---|---|
| **Quick** | Single file, clear intent, <30 min | Just do it. Developer role only. Apply Karpathy principles (surgical, simple, goal-driven). | "Fix the typo in the header", "Add a loading spinner to this button" |
| **Standard** | Multi-file, some ambiguity, 1-4 hours | Plan → Build → Review. Architect + Developer + QA roles. | "Add pagination to the users endpoint", "Refactor auth to use JWT" |
| **Major** | New capability, multiple components, >4 hours or "I'm not sure what to build" | Discover → Plan → Build → Ship. All roles. | "Add a payment system", "Build the analytics dashboard", "We need better onboarding" |

### Detection heuristics

The orchestrator evaluates these — the user doesn't have to classify:

- User explicitly says scope ("quick fix", "big feature", "not sure yet") → trust them
- Request mentions multiple systems/components → Standard or Major
- Request is vague about requirements ("better", "improve", "add support for") → Major
- Request references specific file + specific change → Quick
- User can override: "just do it" skips to build, "let's think about this" escalates to discover

### Slash commands

- `/shipwright` — default entry point, orchestrator classifies and routes
- `/shipwright discover` — force Major lifecycle from discovery phase
- `/shipwright plan` — force Standard lifecycle from planning phase
- `/shipwright build` — force Quick, just implement
- `/shipwright review` — review current changes
- `/shipwright ship` — release workflow

**How `/shipwright` classifies without explicit tier:**
The orchestrator reads the user's description and scores against the heuristics above. If ambiguous, it defaults to Standard (the middle ground) and tells the user why: "This touches multiple files and has some open questions — I'd classify it as Standard and start with a plan. Agree?"

**Key principle:** The orchestrator suggests, never blocks. "This looks like a Standard task — I'd normally plan first then build. Want to do that, or jump straight in?"

---

## 3. The Four Phases

### Phase 1: Discover

**When:** Major tier. The "why" and "what" before the "how."

**Roles involved:** PM (lead), UX (support), Architect (feasibility check)

**Activities:**

1. **Problem framing** (PM) — Why are we building this? What's the user problem? What does success look like? What are we explicitly NOT building?
2. **User flow mapping** (UX) — Who are the users? What's their journey? Where are the friction points? What are the key screens/interactions?
3. **Feasibility gut-check** (Architect) — Is this technically viable? Any obvious blockers? What are the big unknowns?
4. **Scope negotiation** (PM) — Given the flows and feasibility, what's the MVP? What gets cut? What's phase 2?

**Output:** A discovery doc saved to `docs/shipwright/discover/<topic>.md` with:
- Problem statement and success criteria
- User flows (text-based, Mermaid if helpful)
- Scoped feature list (in/out)
- Open questions and risks

**Transition:** Orchestrator suggests "Discovery complete. Ready to plan the implementation?"

### Phase 2: Plan

**When:** Standard and Major tiers.

**Roles involved:** Architect (lead), Developer (implementation reality check)

**Activities:**

1. **Codebase analysis** (Architect) — Read existing patterns, conventions, relevant files. Understand what exists before proposing what to build.
2. **Architecture decisions** (Architect) — Component design, data flow, interfaces. For any non-obvious choice, write an ADR: context, options considered, decision, consequences.
3. **Implementation sequence** (Developer) — Break into ordered steps. Each step should be independently testable. Identify what can be parallelised.
4. **Verification strategy** (Developer + QA preview) — For each step, define: what test proves it works? What could go wrong?

**Output:** An implementation plan saved to `docs/shipwright/plan/<topic>.md` with:
- Architecture decisions (with ADRs for non-trivial ones)
- Ordered task list with verification criteria per step
- Files to create/modify
- Risk flags

**Transition:** Orchestrator suggests "Plan ready. Start building?"

### Phase 3: Build

**When:** All tiers (Quick goes straight here).

**Roles involved:** Developer (lead), QA (continuous)

**Activities:**

1. **TDD loop** (Developer) — Write test → implement → verify → next step. Goal-driven execution: every step has a success criterion.
2. **Surgical implementation** (Developer) — Touch only what the task requires. Match existing style. No drive-by refactoring.
3. **Continuous QA** (QA) — After each meaningful chunk, QA perspective kicks in: edge cases, error paths, "what if the user does X?" Not a separate phase — woven into build.
4. **Parallel execution** (when applicable) — For independent modules identified in the plan, spawn parallel agents. Each gets its own worktree.

**Output:** Working code with tests passing.

**Transition:** Orchestrator suggests "Build complete. Ready to review and ship?"

### Phase 4: Ship

**When:** Standard and Major tiers.

**Roles involved:** QA (lead), Developer (fixes), Release Manager

**Activities:**

1. **Exploratory testing** (QA) — "Try to break this." Not scripted test cases — adversarial thinking. What did the developer NOT think of?
2. **Code review** (multi-perspective) — Correctness, security, performance, simplicity. Spawn parallel review agents if warranted.
3. **Release prep** (Release Manager) — Changelog entry, PR description, deployment notes if applicable. What should reviewers pay attention to?
4. **Retrospective** (all roles) — What did we learn? Anything worth documenting for future work? Auto-saved to learnings.

**Output:** PR-ready branch with changelog and documentation.

---

## 4. The Five Roles

Each role is a composable persona with a distinct voice, checklist, and output format. They are passive definitions — skills load them, not the other way around.

### PM (Product Manager)

**Voice:** Direct, outcome-focused. Asks "why" and "for whom" before "how." Pushes back on scope creep. Thinks in trade-offs, not features.

**Core questions:**
- What problem does this solve?
- Who has this problem? How do they solve it today?
- What does success look like? How would we measure it?
- What's the smallest version that tests the hypothesis?
- What are we explicitly NOT doing?

**Output format:** Problem statement, success criteria, scoped feature list (in/out), user stories if helpful.

**When invoked:** Discover phase lead. Also consulted when scope changes mid-build.

### UX (User Experience Designer)

**Voice:** Empathetic, flow-oriented. Thinks in user journeys, not components. Asks "what does the user see/feel/do at each step?"

**Core questions:**
- Who are the distinct user types?
- What's the happy path? What's the error path?
- Where does the user make decisions? What information do they need?
- What are the accessibility requirements?
- What existing UI patterns should we follow?

**Output format:** User flows (text or Mermaid), screen inventory, interaction notes, accessibility checklist.

**When invoked:** Discover phase support. Also consulted in Plan phase when component design affects user experience.

### Architect

**Voice:** Systematic, trade-off-aware. Thinks in boundaries, interfaces, and consequences. Prefers boring technology.

**Core questions:**
- What are the components and how do they communicate?
- What are the data flows?
- Where are the boundaries/interfaces?
- What's the simplest design that handles the requirements?
- What non-obvious choice am I making, and why? (ADR trigger)

**Output format:** Component diagram (Mermaid), interface contracts, ADRs for non-trivial decisions.

**ADR format:**
```markdown
## ADR: <title>
**Status:** Accepted
**Context:** What situation are we in?
**Options:** What did we consider?
**Decision:** What did we pick and why?
**Consequences:** What follows from this?
```

**When invoked:** Discover feasibility check. Plan phase lead.

### Developer

**Voice:** Pragmatic, Karpathy-disciplined. Writes minimum code that solves the problem. Every changed line traces to the request. Tests first.

**Core principles:**
- State assumptions explicitly before coding
- Simplicity first — no speculative abstractions
- Surgical changes — touch only what you must
- Goal-driven — define verification before implementing
- TDD — test → implement → verify

**Output format:** Working code, tests, commit-ready changes.

**When invoked:** Plan phase reality check. Build phase lead.

### QA (Quality Assurance)

**Voice:** Adversarial, curious. "What if?" mindset. Not malicious — genuinely trying to find where things break before users do.

**Core questions:**
- What happens with empty input? Null? Huge input?
- What if this is called twice? Concurrently?
- What if the network fails? The database is slow?
- What does the user see when things go wrong?
- What assumption would be most expensive if wrong?

**Activities:**
- Edge case generation — systematic, not random
- Exploratory testing — "use it like a confused user would"
- Security surface check — injection, auth bypass, data leaks
- Performance smell check — obvious N+1s, unbounded queries, missing indexes

**Output format:** Issues found (severity + reproduction steps), confidence assessment, "ship it" or "fix these first" recommendation.

**When invoked:** Build phase continuous companion. Ship phase lead.

---

## 5. Project Structure

```
shipwright/
├── .claude-plugin                    # Plugin manifest
├── CLAUDE.md                         # Karpathy principles + Shipwright identity
├── README.md                         # Docs, installation, credits
├── LICENSE                           # MIT
├── skills/
│   ├── orchestrate/
│   │   └── SKILL.md                  # Entry point — complexity detection, phase routing
│   ├── discover/
│   │   └── SKILL.md                  # Phase 1 — PM-led discovery
│   ├── plan/
│   │   └── SKILL.md                  # Phase 2 — Architect-led planning
│   ├── build/
│   │   └── SKILL.md                  # Phase 3 — Developer-led implementation
│   ├── ship/
│   │   └── SKILL.md                  # Phase 4 — QA-led release
│   └── roles/
│       ├── pm.md                     # PM persona definition
│       ├── ux.md                     # UX persona definition
│       ├── architect.md              # Architect persona definition
│       ├── developer.md              # Developer persona definition
│       └── qa.md                     # QA persona definition
├── agents/
│   ├── review/
│   │   ├── correctness.md            # Code correctness reviewer
│   │   ├── security.md               # Security surface reviewer
│   │   ├── simplicity.md             # Karpathy simplicity reviewer
│   │   └── performance.md            # Performance smell detector
│   ├── qa/
│   │   ├── edge-case-hunter.md       # Systematic edge case finder
│   │   └── exploratory-tester.md     # "Use it like a confused user"
│   └── research/
│       ├── codebase-analyst.md       # Existing patterns & conventions
│       └── docs-researcher.md        # External docs & best practices
└── docs/
    └── methodology.md                # The full methodology explained
```

### How skills load roles

Each phase skill reads its role files at invocation. For example, `discover/SKILL.md` starts with:

```markdown
Load and apply these role perspectives for this phase:
- roles/pm.md (lead)
- roles/ux.md (support)
- roles/architect.md (feasibility check)
```

This keeps roles DRY — update the PM voice once, every phase that uses PM gets the update.

### How the orchestrator works

`/shipwright` (or `/shipwright <description>`) invokes `orchestrate/SKILL.md`, which:
1. Reads the user's request
2. Classifies tier (Quick/Standard/Major)
3. Announces: "This looks like a [tier] task. I'd recommend [phase]. Want to proceed, or adjust?"
4. On confirmation, invokes the appropriate phase skill

### Where user projects store Shipwright artifacts

```
docs/shipwright/
├── discover/          # Discovery docs
├── plan/              # Implementation plans + ADRs
├── learn/             # Retrospective learnings
└── changelog.md       # Release changelog
```

---

## 6. Key Design Decisions

### 1. Skills read role files, roles don't invoke skills

Roles are passive definitions (voice, checklists, questions). Skills are active workflows. You can't `/shipwright pm` — there's no PM skill. You `/shipwright discover`, and the discover skill loads the PM role. This prevents people from asking "the PM" to write code or "the QA" to do architecture. Roles serve phases, not the other way around.

### 2. The orchestrator suggests, never blocks

Unlike superpowers which gates implementation behind brainstorming, Shipwright always lets you proceed. "This looks Major — I'd start with discovery. Want to, or jump straight to build?" If you say "just build it", it builds it. The methodology earns trust through results, not enforcement.

### 3. ADRs are lightweight and inline

No separate ADR directory or tooling. When the architect role encounters a non-obvious choice during planning, it writes a short ADR block directly in the plan document. Context, options, decision, consequences — four lines, not a ceremony.

### 4. QA is continuous, not a gate

QA doesn't wait for "build complete" to start. During the build phase, the QA role is a companion — after each meaningful chunk, it asks its questions. This catches issues during construction, not after.

### 5. Artifacts live in the user's repo, not the plugin

Discovery docs, plans, ADRs, learnings — all saved to `docs/shipwright/` in the user's project. Version-controlled, reviewable, persistent across sessions. The plugin is stateless.

### 6. No framework opinions

Shipwright never says "use React" or "use PostgreSQL." The developer and architect roles read the existing codebase and follow its patterns. The methodology is about *how* to think, not *what* to use.

---

## 7. Phased Release Plan

| Phase | Contents | Focus |
|---|---|---|
| **P1** | Orchestrator, Discover, Plan, roles (PM, UX, Architect) | The novel stuff — product and design thinking |
| **P2** | Build, Ship, roles (Developer, QA), review agents | Full lifecycle — implementation and quality |
| **P3** | Research agents, learning loops, multi-agent orchestration | Advanced — compounding and parallelism |

---

*This is the design specification for Shipwright. Implementation plans follow separately per phase.*
